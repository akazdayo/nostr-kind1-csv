<script lang="ts">
	import { SimplePool } from 'nostr-tools/pool';
	import * as nip19 from 'nostr-tools/nip19';
	import type { Event } from 'nostr-tools/core';

	let npubInput = $state('');
	let relaysInput = $state('wss://yabu.me');
	let csvOutput = $state('');
	let isLoading = $state(false);
	let errorMessage = $state('');
	let successMessage = $state('');

	const CHUNK_EVENT_LIMIT = 500;
	const CHUNK_TIMEOUT_MS = 10_000;

	type Kind1Filter = {
		kinds: number[];
		authors: string[];
		limit?: number;
		until?: number;
	};

	// CSVエスケープ関数
	function escapeCSV(value: string): string {
		if (value.includes('"') || value.includes(',') || value.includes('\n')) {
			return `"${value.replace(/"/g, '""')}"`;
		}
		return value;
	}

	// npubをhex形式に変換
	function npubToHex(npub: string): string {
		try {
			const decoded = nip19.decode(npub.trim());
			if (decoded.type === 'npub') {
				return decoded.data;
			}
			throw new Error('Invalid npub format');
		} catch (error) {
			throw new Error(`Failed to decode npub: ${error instanceof Error ? error.message : 'Unknown error'}`);
		}
	}

	// リレーURLをパース
	function parseRelays(input: string): string[] {
		return input
			.split(/[\n,]/)
			.map((url) => url.trim())
			.filter((url) => url.length > 0);
	}

	async function fetchChunkFromRelays(
		pool: SimplePool,
		relays: string[],
		filter: Kind1Filter,
		timeoutMs = CHUNK_TIMEOUT_MS
	): Promise<Event[]> {
		return new Promise((resolve) => {
			const chunkEvents: Event[] = [];
			const subscription = pool.subscribeMany(relays, filter, {
				onevent(event) {
					chunkEvents.push(event);
				},
				oneose() {
					cleanup();
					resolve(chunkEvents);
				},
				onclose() {
					cleanup();
					resolve(chunkEvents);
				}
			});

			const timeoutId = setTimeout(() => {
				cleanup();
				resolve(chunkEvents);
			}, timeoutMs);

			function cleanup() {
				clearTimeout(timeoutId);
				void subscription?.close();
			}
		});
	}

	// Nostrイベントを取得してCSVに変換
	async function fetchAndConvertToCSV() {
		errorMessage = '';
		successMessage = '';
		csvOutput = '';
		isLoading = true;

		try {
			// npubをhexに変換
			const pubkeyHex = npubToHex(npubInput);

			// リレーURLをパース
			const relays = parseRelays(relaysInput);
			if (relays.length === 0) {
				throw new Error('At least one relay URL is required');
			}

			// SimplePoolを使用してイベントを取得
			const pool = new SimplePool();
			const events: Event[] = [];
			const seenEventIds = new Set<string>();
			let until: number | undefined;

			try {
				while (true) {
					const filter: Kind1Filter = {
						kinds: [1],
						authors: [pubkeyHex],
						limit: CHUNK_EVENT_LIMIT
					};

					if (typeof until === 'number') {
						filter.until = until;
					}

					const chunk = await fetchChunkFromRelays(pool, relays, filter);

					if (chunk.length === 0) {
						break;
					}

					let oldestTimestamp = chunk[0].created_at;

					for (const event of chunk) {
						if (event.created_at < oldestTimestamp) {
							oldestTimestamp = event.created_at;
						}

						if (!seenEventIds.has(event.id)) {
							seenEventIds.add(event.id);
							events.push(event);
						}
					}

					if (chunk.length < CHUNK_EVENT_LIMIT) {
						break;
					}

					until = oldestTimestamp - 1;

					if (until < 0) {
						break;
					}
				}
			} finally {
				pool.close(relays);
			}

			// イベントをタイムスタンプでソート（古い順）
			events.sort((a, b) => a.created_at - b.created_at);

			// CSVに変換
			const csvLines = ['content,timestamp'];
			for (const event of events) {
				const content = escapeCSV(event.content);
				const timestamp = new Date(event.created_at * 1000).toISOString();
				csvLines.push(`${content},${timestamp}`);
			}

			csvOutput = csvLines.join('\n');
			isLoading = false;

			if (events.length === 0) {
				successMessage = 'No kind1 events found for this user';
			} else {
				successMessage = `Successfully fetched ${events.length} kind1 events`;
			}
		} catch (error) {
			isLoading = false;
			errorMessage = error instanceof Error ? error.message : 'An unknown error occurred';
		}
	}

	// クリップボードにコピー
	async function copyToClipboard() {
		try {
			await navigator.clipboard.writeText(csvOutput);
			successMessage = 'Copied to clipboard!';
			setTimeout(() => {
				if (successMessage === 'Copied to clipboard!') {
					successMessage = '';
				}
			}, 2000);
		} catch (error) {
			errorMessage = 'Failed to copy to clipboard';
		}
	}
</script>

<div class="min-h-screen bg-base-200 p-4">
	<div class="container mx-auto max-w-4xl">
		<div class="card bg-base-100 shadow-xl">
			<div class="card-body">
				<h1 class="card-title text-3xl mb-4">Nostr Kind1 CSV Exporter</h1>

				<!-- エラーメッセージ -->
				{#if errorMessage}
					<div class="alert alert-error">
						<svg
							xmlns="http://www.w3.org/2000/svg"
							class="stroke-current shrink-0 h-6 w-6"
							fill="none"
							viewBox="0 0 24 24"
						>
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								stroke-width="2"
								d="M10 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z"
							/>
						</svg>
						<span>{errorMessage}</span>
					</div>
				{/if}

				<!-- 成功メッセージ -->
				{#if successMessage}
					<div class="alert alert-success">
						<svg
							xmlns="http://www.w3.org/2000/svg"
							class="stroke-current shrink-0 h-6 w-6"
							fill="none"
							viewBox="0 0 24 24"
						>
							<path
								stroke-linecap="round"
								stroke-linejoin="round"
								stroke-width="2"
								d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"
							/>
						</svg>
						<span>{successMessage}</span>
					</div>
				{/if}

				<!-- 入力フォーム -->
				<div class="form-control w-full">
					<label for="npub-input" class="label">
						<span class="label-text font-semibold">Public Key (npub)</span>
					</label>
					<input
						id="npub-input"
						type="text"
						placeholder="npub1..."
						class="input input-bordered w-full"
						bind:value={npubInput}
					/>
				</div>

				<div class="form-control w-full mt-4">
					<label for="relays-input" class="label">
						<span class="label-text font-semibold">Relays (one per line or comma-separated)</span>
					</label>
					<textarea
						id="relays-input"
						class="textarea textarea-bordered h-24"
						placeholder="wss://relay.example.com"
						bind:value={relaysInput}
					></textarea>
				</div>

				<!-- 取得ボタン -->
				<div class="card-actions justify-end mt-4">
					<button
						class="btn btn-primary"
						onclick={fetchAndConvertToCSV}
						disabled={isLoading || !npubInput.trim()}
					>
						{#if isLoading}
							<span class="loading loading-spinner"></span>
							Fetching...
						{:else}
							Fetch Events
						{/if}
					</button>
				</div>

				<!-- CSV出力エリア -->
				{#if csvOutput}
					<div class="divider"></div>

					<div class="form-control w-full">
						<div class="label">
							<span class="label-text font-semibold">CSV Output</span>
							<button class="btn btn-sm btn-outline" onclick={copyToClipboard}>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									class="h-4 w-4"
									fill="none"
									viewBox="0 0 24 24"
									stroke="currentColor"
								>
									<path
										stroke-linecap="round"
										stroke-linejoin="round"
										stroke-width="2"
										d="M8 16H6a2 2 0 01-2-2V6a2 2 0 012-2h8a2 2 0 012 2v2m-6 12h8a2 2 0 002-2v-8a2 2 0 00-2-2h-8a2 2 0 00-2 2v8a2 2 0 002 2z"
									/>
								</svg>
								Copy
							</button>
						</div>
						<textarea
							class="textarea textarea-bordered font-mono text-sm h-96"
							readonly
							value={csvOutput}
						></textarea>
					</div>
				{/if}
			</div>
		</div>
	</div>
</div>
