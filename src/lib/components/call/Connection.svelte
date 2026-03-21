<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { socket } from '$lib/stores';

	export let channelId = '';
	export let enableVideo = true;
	export let enableAudio = true;

	let localStream: MediaStream | null = null;
	let peers: Map<string, { pc: RTCPeerConnection; stream: MediaStream }> = new Map();
	let remoteStreams: MediaStream[] = [];
	let localVideoEl: HTMLVideoElement;

	const srcObject = (node: HTMLMediaElement, stream: MediaStream) => {
		node.srcObject = stream;
		node.play().catch((e) => console.warn('[call] autoplay blocked:', e.message));
		return {
			update(newStream: MediaStream) {
				if (node.srcObject !== newStream) {
					node.srcObject = newStream;
					node.play().catch((e) => console.warn('[call] autoplay blocked:', e.message));
				}
			}
		};
	};

	$: if (localStream) {
		localStream.getAudioTracks().forEach((t) => (t.enabled = enableAudio));
		localStream.getVideoTracks().forEach((t) => (t.enabled = enableVideo));
	}

	const updateRemoteStreams = () => {
		remoteStreams = [...peers.values()].map((p) => p.stream);
	};

	const createPC = (peerId: string): RTCPeerConnection => {
		const pc = new RTCPeerConnection();
		const stream = new MediaStream();
		peers.set(peerId, { pc, stream });

		// add local tracks
		if (localStream) {
			for (const track of localStream.getTracks()) {
				pc.addTrack(track, localStream);
			}
		}

		pc.ontrack = (e) => {
			console.log('[call] track from', peerId, e.track.kind);
			stream.addTrack(e.track);
			updateRemoteStreams();
		};

		pc.onicecandidate = (e) => {
			if (e.candidate) {
				$socket?.emit('events:call', {
					channel_id: channelId,
					type: 'candidate',
					to: peerId,
					data: e.candidate.toJSON()
				});
			}
		};

		pc.oniceconnectionstatechange = () => {
			console.log('[call]', peerId, 'ice:', pc.iceConnectionState);
		};

		return pc;
	};

	const handleCallEvent = async (event: any) => {
		if (event.channel_id !== channelId) return;

		const type = event.type;

		if (type === 'peer-joined') {
			// a new peer entered — we initiate the offer
			const peerId = event.peer_id;
			console.log('[call] peer joined, creating offer for', peerId);
			const pc = createPC(peerId);
			const offer = await pc.createOffer();
			await pc.setLocalDescription(offer);
			$socket?.emit('events:call', {
				channel_id: channelId,
				type: 'offer',
				to: peerId,
				data: { sdp: offer.sdp, type: offer.type }
			});
		} else if (type === 'offer') {
			// a peer sent us an offer — create PC, set remote, send answer
			const peerId = event.from;
			console.log('[call] received offer from', peerId);
			const pc = createPC(peerId);
			await pc.setRemoteDescription(new RTCSessionDescription(event.data));
			const answer = await pc.createAnswer();
			await pc.setLocalDescription(answer);
			$socket?.emit('events:call', {
				channel_id: channelId,
				type: 'answer',
				to: peerId,
				data: { sdp: answer.sdp, type: answer.type }
			});
		} else if (type === 'answer') {
			const peerId = event.from;
			const peer = peers.get(peerId);
			if (peer) {
				console.log('[call] received answer from', peerId);
				await peer.pc.setRemoteDescription(new RTCSessionDescription(event.data));
			}
		} else if (type === 'candidate') {
			const peerId = event.from;
			const peer = peers.get(peerId);
			if (peer) {
				await peer.pc.addIceCandidate(new RTCIceCandidate(event.data));
			}
		} else if (type === 'peer-left') {
			const peerId = event.peer_id;
			const peer = peers.get(peerId);
			if (peer) {
				console.log('[call] peer left:', peerId);
				peer.pc.close();
				peers.delete(peerId);
				updateRemoteStreams();
			}
		}
	};

	const connect = async () => {
		console.log('[call] joining channel:', channelId);

		localStream = await navigator.mediaDevices.getUserMedia({
			video: enableVideo,
			audio: enableAudio
		});

		if (localVideoEl) {
			localVideoEl.srcObject = localStream;
		}

		$socket?.on('events:call', handleCallEvent);

		// tell the server we've joined — existing peers will send us offers
		$socket?.emit('events:call', {
			channel_id: channelId,
			type: 'join'
		});
	};

	const cleanup = () => {
		console.log('[call] leaving');
		$socket?.emit('events:call', { channel_id: channelId, type: 'leave' });
		$socket?.off('events:call', handleCallEvent);

		for (const [, peer] of peers) {
			peer.pc.close();
		}
		peers.clear();
		remoteStreams = [];

		localStream?.getTracks().forEach((t) => t.stop());
		localStream = null;
	};

	onMount(() => connect());
	onDestroy(() => cleanup());
</script>

<div class="relative w-full h-full bg-gray-950 flex flex-col">
	<!-- remote streams grid -->
	<div class="flex-1 min-h-0 flex items-center justify-center p-4">
		{#if remoteStreams.length === 0}
			<div class="text-gray-500 text-sm">Waiting for others to join...</div>
		{:else}
			<div
				class="grid gap-3 w-full h-full place-items-center"
				style="grid-template-columns: repeat({Math.min(remoteStreams.length, 3)}, 1fr); grid-template-rows: repeat({Math.ceil(remoteStreams.length / 3)}, 1fr);"
			>
				{#each remoteStreams as stream (stream.id)}
					<div class="relative w-full h-full flex items-center justify-center">
						<video
							class="max-w-full max-h-full w-auto h-auto rounded-xl bg-gray-900 shadow-lg"
							autoplay
							playsinline
							use:srcObject={stream}
						></video>
					</div>
				{/each}
			</div>
		{/if}
	</div>

	<!-- local preview -->
	<div class="absolute bottom-20 right-4 w-44 rounded-xl overflow-hidden shadow-xl border border-gray-700/50 bg-gray-900">
		<video
			class="w-full aspect-video object-cover"
			autoplay
			playsinline
			muted
			bind:this={localVideoEl}
		></video>
	</div>
</div>
