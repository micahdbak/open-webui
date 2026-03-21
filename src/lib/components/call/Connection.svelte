<script lang="ts">
	import { onMount, onDestroy, tick } from 'svelte';
	import { socket } from '$lib/stores';
	import { toast } from 'svelte-sonner';
	import Spinner from '$lib/components/common/Spinner.svelte';

	export let channelId = '';
	export let enableVideo = true;
	export let enableAudio = true;

	let localStream: MediaStream | null = null;
	let pc: RTCPeerConnection | null = null;
	let remoteVideoStreams: MediaStream[] = []; // video-only streams, one per remote peer
	let remoteAudioStream: MediaStream = new MediaStream(); // combined audio from all peers

	// peer track correlation: renegotiate populates the queue, ontrack dequeues and maps
	let pendingTrackPeerIds: string[] = [];
	let peerIdToVideoStreams: Record<string, MediaStream[]> = {};
	let peerIdToAudioTracks: Record<string, MediaStreamTrack[]> = {};

	let serverRenegotiating = false; // suppress onnegotiationneeded during server-driven renegotiation
	let connected = false; // true once initial signaling completes

	let localVideoEl: HTMLVideoElement;
	let remoteAudioEl: HTMLAudioElement;
	let remoteVideoEls: HTMLVideoElement[] = [];

	// sync local preview
	$: if (localVideoEl && localStream) {
		if (localVideoEl.srcObject !== localStream) {
			localVideoEl.srcObject = localStream;
			localVideoEl.play().catch(() => {});
		}
	}

	// sync remote audio
	$: if (remoteAudioEl && remoteAudioStream) {
		if (remoteAudioEl.srcObject !== remoteAudioStream) {
			remoteAudioEl.srcObject = remoteAudioStream;
			remoteAudioEl.play().catch(() => {});
		}
	}

	// sync remote video elements — wait for DOM to flush so bind:this has run
	$: (remoteVideoStreams, tick().then(syncRemoteVideos));
	const syncRemoteVideos = () => {
		remoteVideoEls.forEach((el, i) => {
			if (el && remoteVideoStreams[i] && el.srcObject !== remoteVideoStreams[i]) {
				el.srcObject = remoteVideoStreams[i];
				el.play().catch(() => {});
			}
		});
	};

	$: if (localStream) {
		localStream.getAudioTracks().forEach((t) => (t.enabled = enableAudio));
		localStream.getVideoTracks().forEach((t) => (t.enabled = enableVideo));
	}

	const emitOffer = (offer: RTCSessionDescriptionInit) => {
		$socket?.emit('channel:call:offer', {
			channel_id: channelId,
			data: {
				sdp: offer.sdp,
				type: offer.type
			}
		});
	};

	const initHandler = async () => {
		if (pc) {
			pc.close();
			pc = null;
		}

		remoteVideoStreams = [];
		remoteAudioStream = new MediaStream();

		try {
			localStream = await navigator.mediaDevices.getUserMedia({
				video: enableVideo,
				audio: enableAudio
			});
		} catch (err: any) {
			if (err.name === 'NotAllowedError') {
				toast.error('Camera/microphone permission was denied');
			} else if (err.name === 'NotFoundError') {
				toast.error('No camera or microphone found');
			} else {
				toast.error(`Could not access media devices: ${err.message}`);
			}
			return;
		}

		console.log(
			'[call] creating connection with local media:',
			localStream.getTracks().map((t) => t.kind)
		);

		pc = new RTCPeerConnection();

		pc.ontrack = (e) => {
			const peerId = pendingTrackPeerIds.shift();

			if (e.track.kind === 'video') {
				const stream = new MediaStream([e.track]);
				remoteVideoStreams = [...remoteVideoStreams, stream];
				if (peerId) {
					peerIdToVideoStreams[peerId] = [...(peerIdToVideoStreams[peerId] || []), stream];
				}
				console.log(`[call] received video stream${peerId ? ` from peer ${peerId}` : ''}`);
			} else if (e.track.kind === 'audio') {
				remoteAudioStream.addTrack(e.track);
				remoteAudioStream = remoteAudioStream; // trigger reactivity
				if (peerId) {
					peerIdToAudioTracks[peerId] = [...(peerIdToAudioTracks[peerId] || []), e.track];
				}
				console.log(`[call] received audio stream${peerId ? ` from peer ${peerId}` : ''}`);
			}
		};

		pc.onconnectionstatechange = () => {
			const state = pc?.connectionState;
			console.log(`[call] connection state: ${state}`);
			if (state === 'failed') {
				toast.error('Call connection failed');
			} else if (state === 'disconnected') {
				toast.error('Call connection lost');
			}
		};

		pc.onnegotiationneeded = async () => {
			try {
				if (serverRenegotiating) return;
				if (pc?.signalingState !== 'stable') return;

				const offer = await pc.createOffer();
				await pc.setLocalDescription(offer);
				emitOffer(offer);
			} catch (err: any) {
				console.error('[call] negotiation error:', err);
				toast.error('Call negotiation failed');
			}
		};

		// will trigger negotiationneeded
		for (const track of localStream.getTracks()) {
			pc.addTrack(track, localStream);
		}
	};

	const leave = () => {
		console.log('[call] leaving');

		$socket?.emit('channel:call:leave', {
			channel_id: channelId
		});

		localStream?.getTracks().forEach((t) => t.stop());
		localStream = null;

		pc?.close();
		pc = null;

		remoteVideoStreams = [];
	};

	const handleAnswer = async (event: any) => {
		if (event.channel_id !== channelId) return;

		console.log('[call] received answer');

		if (pc?.signalingState !== 'have-local-offer') {
			console.log('[call] ignoring answer, not in have-local-offer state');
			return;
		}

		try {
			await pc.setRemoteDescription(new RTCSessionDescription(event.data));
			serverRenegotiating = false;
			connected = true;
		} catch (err: any) {
			console.error('[call] failed to set remote description:', err);
			toast.error('Failed to establish call connection');
		}
	};

	const handleRenegotiate = async (event: any) => {
		if (event.channel_id !== channelId || !pc) return;

		console.log('[call] received renegotiate');

		if (pc.signalingState !== 'stable') {
			console.log('[call] ignoring renegotiate, not in stable state');
			return;
		}

		try {
			// add recvonly transceivers and queue peer IDs for ontrack correlation
			serverRenegotiating = true;
			const requestedTransceivers = event.data || [];
			for (const { peer_id, kind } of requestedTransceivers) {
				pc.addTransceiver(kind as string, { direction: 'recvonly' });
				pendingTrackPeerIds.push(peer_id);
			}

			const offer = await pc.createOffer();
			await pc.setLocalDescription(offer);
			emitOffer(offer);
		} catch (err: any) {
			console.error('[call] renegotiation error:', err);
			toast.error('Failed to add peer to call');
			serverRenegotiating = false;
		}
	};

	const handlePeerLeft = async (event: any) => {
		if (event.channel_id !== channelId) return;

		const peerId = event.data?.peer_id;
		if (!peerId) return;

		console.log(`[call] peer ${peerId} left, cleaning up`);

		// remove video streams belonging to this peer
		const peerStreams = peerIdToVideoStreams[peerId] || [];
		if (peerStreams.length > 0) {
			remoteVideoStreams = remoteVideoStreams.filter((s) => !peerStreams.includes(s));
			delete peerIdToVideoStreams[peerId];
		}

		// remove audio tracks belonging to this peer
		const peerAudioTracks = peerIdToAudioTracks[peerId] || [];
		for (const track of peerAudioTracks) {
			remoteAudioStream.removeTrack(track);
		}
		if (peerAudioTracks.length > 0) {
			remoteAudioStream = remoteAudioStream; // trigger reactivity
			delete peerIdToAudioTracks[peerId];
		}
	};

	onMount(() => {
		$socket?.on('channel:call:answer', handleAnswer);
		$socket?.on('channel:call:renegotiate', handleRenegotiate);
		$socket?.on('channel:call:peer-left', handlePeerLeft);
		initHandler();
	});

	onDestroy(() => {
		$socket?.off('channel:call:answer', handleAnswer);
		$socket?.off('channel:call:renegotiate', handleRenegotiate);
		$socket?.off('channel:call:peer-left', handlePeerLeft);
		leave();
	});
</script>

<div class="relative w-full h-full bg-gray-950 flex flex-col">
	<!-- remote streams grid -->
	<div class="flex-1 min-h-0 flex items-center justify-center p-4">
		{#if !connected}
			<div class="flex flex-col items-center gap-3">
				<Spinner className="size-6 text-gray-400" />
			</div>
		{:else if remoteVideoStreams.length === 0}
			<div class="text-gray-500 text-sm">You're alone here.</div>
		{:else}
			<div
				class="grid gap-3 w-full h-full place-items-center"
				style="grid-template-columns: repeat({Math.min(
					remoteVideoStreams.length,
					3
				)}, 1fr); grid-template-rows: repeat({Math.ceil(remoteVideoStreams.length / 3)}, 1fr);"
			>
				{#each remoteVideoStreams as stream, i (stream.id)}
					<div class="relative w-full h-full flex items-center justify-center">
						<video
							class="max-w-full max-h-full w-auto h-auto rounded-xl bg-gray-900 shadow-lg"
							autoplay
							playsinline
							muted
							bind:this={remoteVideoEls[i]}
						></video>
					</div>
				{/each}
			</div>
		{/if}
	</div>

	<!-- hidden audio element for all remote audio -->
	<!-- svelte-ignore a11y-media-has-caption -->
	<audio autoplay bind:this={remoteAudioEl}></audio>

	<!-- local preview overlay -->
	<div
		class="absolute bottom-20 right-4 w-44 rounded-xl overflow-hidden shadow-xl border border-gray-700/50 bg-gray-900"
	>
		<video
			class="w-full aspect-video object-cover"
			autoplay
			playsinline
			muted
			bind:this={localVideoEl}
		></video>
	</div>
</div>
