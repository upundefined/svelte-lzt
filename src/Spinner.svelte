<script lang="ts">
	interface Props {
		type: 'medium' | 'small';
		variant: 'logoForum' | 'logoMarket' | 'bounce';
		inline?: boolean;
	}

	let { variant, type, inline = false }: Props = $props();
</script>

<div class="lztui-spinner-wrapper" class:lztui-spinner-full={!inline}>
	{#if variant === 'logoForum'}
		<div class="lztui-spinner-logo">
			<svg class={type} xmlns="http://www.w3.org/2000/svg" width="256" height="256" viewBox="0 0 256 256" fill="none"
				><defs
					><linearGradient id="shimmer"
						><stop offset="0%" stop-color="#393939"><animate attributeName="offset" values="-2;-1;1;2" dur="1.5s" repeatCount="indefinite" /></stop><stop offset="50%" stop-color="#505050"><animate attributeName="offset" values="-1;0;2;3" dur="1.5s" repeatCount="indefinite" /></stop><stop offset="100%" stop-color="#393939"
							><animate attributeName="offset" values="0;1;3;4" dur="1.5s" repeatCount="indefinite" /></stop
						></linearGradient
					><clipPath id="clip0_123_378"><rect width="236.512" height="214.598" fill="white" transform="translate(9.4707 20.4229)" /></clipPath></defs
				><g clip-path="url(#clip0_123_378)"
					><path
						fill-rule="evenodd"
						clip-rule="evenodd"
						d="M70.1504 207.392C91.8146 224.992 116.842 235.022 143.533 235.022C183.225 235.022 219.257 212.775 245.995 176.542C234.706 161.247 221.764 148.436 207.575 138.751L184.794 150.124C189.69 157.741 192.542 166.809 192.542 176.542C192.542 203.577 170.604 225.491 143.533 225.491C122.499 225.491 104.566 212.252 97.6139 193.666L70.1504 207.38V207.392ZM126.67 179.168C127.216 182.721 128.88 186.025 131.459 188.604C134.656 191.8 139.005 193.595 143.533 193.583C152.957 193.583 160.598 185.953 160.598 176.542C160.598 171.919 158.756 167.724 155.761 164.646L126.682 179.168H126.67Z"
						fill="url(#shimmer)"
					/><path d="M76.3528 176.16L242.405 96.3367L205.91 20.4229L187.062 87.507L164.412 40.3758L145.552 107.46L122.902 60.3287L104.042 127.413L81.3916 80.2816L62.5319 147.366L39.8814 100.235L9.4707 208.318L50.9809 188.365L76.3528 176.172V176.16Z" fill="url(#shimmer)" /></g
				></svg
			>
		</div>
	{:else if variant === 'logoMarket'}
		<div class="lztui-spinner-logo">
			<svg class={type} xmlns="http://www.w3.org/2000/svg" width="256" height="256" viewBox="0 0 256 256" fill="none"
				><defs
					><!-- Анимированный шиммер-градиент --><linearGradient id="shimmer"
						><stop offset="0%" stop-color="#393939"><animate attributeName="offset" values="-2;-1;1;2" dur="1.5s" repeatCount="indefinite" /></stop><stop offset="50%" stop-color="#505050"><animate attributeName="offset" values="-1;0;2;3" dur="1.5s" repeatCount="indefinite" /></stop><stop offset="100%" stop-color="#393939"
							><animate attributeName="offset" values="0;1;3;4" dur="1.5s" repeatCount="indefinite" /></stop
						></linearGradient
					></defs
				><path
					d="M4 40C28.2591 85.11 70.8437 98.4849 122.833 99.5138C153.51 100.147 210.666 97.6539 210.666 113.759C210.666 125.867 176.438 157.286 176.438 165.042C176.438 165.042 207.479 139.4 224.876 124.206C242.271 109.011 252 99.949 252 90.9665C252 72.6851 179.828 76.0881 121.542 76.0881C63.2552 76.0881 31.4076 65.1666 4 40ZM66 174.539C54.5769 174.539 45.3334 183.601 45.3334 194.799C45.3334 205.997 54.5769 215.059 66 215.059C77.4231 215.059 86.6666 205.997 86.6666 194.799C86.6666 183.601 77.4231 174.539 66 174.539ZM148.667 174.539C137.243 174.539 128 183.601 128 194.799C128 205.997 137.243 215.059 148.667 215.059C160.09 215.059 169.333 205.997 169.333 194.799C169.333 183.601 160.09 174.539 148.667 174.539Z"
					fill="url(#shimmer)"
				/></svg
			>
		</div>
	{:else}
		<div class="lztui-spinner lztui-spinner-{type}" class:lztui-spinner-full={!inline}>
			<div class="lztui-spinner-bounce lztui-spinner-bounce1"></div>
			<div class="lztui-spinner-bounce lztui-spinner-bounce2"></div>
			<div class="lztui-spinner-bounce lztui-spinner-bounce3"></div>
		</div>
	{/if}
</div>

<style>
	.lztui-spinner {
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.lztui-spinner-logo .small {
		width: 28px;
		height: 28px;
	}

	.lztui-spinner-logo .medium {
		width: 40px;
		height: 40px;
	}

	.lztui-spinner-bounce {
		animation: sk-bouncedelay 1.4s infinite ease-in-out both;
		background-color: #454545;
		border-radius: 100%;
		display: inline-block;
		transform-origin: center bottom;
	}

	.lztui-spinner-small .lztui-spinner-bounce {
		width: 6px;
		height: 6px;
		margin: 0 2px;
	}

	.lztui-spinner-medium .lztui-spinner-bounce {
		width: 16px;
		height: 16px;
	}

	.lztui-spinner-bounce1 {
		animation-delay: -0.32s;
	}
	.lztui-spinner-bounce2 {
		animation-delay: -0.16s;
	}

	@keyframes lztui-spinner-bounce {
		0%,
		80%,
		100% {
			transform: scale(0);
		}
		40% {
			transform: scale(1);
		}
	}

	.lztui-spinner-full {
		height: 100%;
	}

	.lztui-spinner-wrapper {
		display: inline-block;
	}

	.lztui-spinner-full.lztui-spinner-wrapper {
		display: flex;
		align-items: center;
	}
</style>
