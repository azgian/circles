<script lang="ts">
	import { onMount } from 'svelte';
	import Circle from './Circle.svelte';

	interface CircleData {
		id: number;
		num: number;
		size: number;
		x: number;
		y: number;
		dx: number;
		dy: number;
		color: string;
		ignoreCollision: number; // 충돌 무시 카운터 추가
	}

	let gameState: 'idle' | 'running' = 'idle';
	let circles: CircleData[] = [];

	let container: HTMLElement;
	let containerWidth: number;
	let containerHeight: number;

	const colors = ['#e60000', '#e67300', '#e6e600', '#00b300', '#0000e6', '#4b0082', '#8600b3'];

	let nextId = 0;
	const getNextId = () => nextId++;

	let timer = 0;
	let gameRunning = false;
	let animationId: number | null = null;
	let timerInterval: number | null = null;

	onMount(() => {
		containerWidth = container.clientWidth;
		containerHeight = container.clientHeight;
		window.addEventListener('resize', handleResize);

		return () => {
			window.removeEventListener('resize', handleResize);
			stopGame();
		};
	});

	const startGame = () => {
		if (gameState === 'idle') {
			gameState = 'running';
			initializeCircles();
			gameRunning = true;
			timer = 0;
			animate();
			startTimer();
		}
	};

	const resetGame = () => {
		stopGame();
		circles = [];
		gameState = 'idle';
		timer = 0;
	};

	const stopGame = () => {
		gameRunning = false;
		if (animationId !== null) {
			cancelAnimationFrame(animationId);
			animationId = null;
		}
		if (timerInterval !== null) {
			clearInterval(timerInterval);
			timerInterval = null;
		}
	};

	const startTimer = () => {
		if (timerInterval === null) {
			timerInterval = setInterval(() => {
				if (gameRunning) {
					timer++;
				} else {
					stopGame();
				}
			}, 1000);
		}
	};

	const handleResize = () => {
		containerWidth = container.clientWidth;
		containerHeight = container.clientHeight;
		initializeCircles();
	};

	const initializeCircles = () => {
		const circleSize = Math.min(containerWidth, containerHeight) * 0.15;
		circles = Array(8)
			.fill(null)
			.map(() => {
				const num = Math.floor(Math.random() * 100);
				return {
					id: getNextId(),
					num,
					size: circleSize,
					x: Math.random() * (containerWidth - circleSize) + circleSize / 2,
					y: Math.random() * (containerHeight - circleSize) + circleSize / 2,
					dx: (Math.random() - 0.5) * 4,
					dy: (Math.random() - 0.5) * 4,
					color: colors[num % 7],
					ignoreCollision: 0
				};
			});
	};

	const checkCollision = (circle1: CircleData, circle2: CircleData) => {
		if (circle1.ignoreCollision > 0 || circle2.ignoreCollision > 0) {
			return [circle1, circle2]; // 충돌 무시 중이면 그대로 반환
		}

		const dx = circle2.x - circle1.x;
		const dy = circle2.y - circle1.y;
		const distance = Math.sqrt(dx * dx + dy * dy);

		if (distance < circle1.size) {
			if (circle1.color === circle2.color) {
				// 같은 색깔의 공들이 충돌한 경우
				const biggerCircle = circle1.num > circle2.num ? { ...circle1 } : { ...circle2 };
				const smallerCircle = circle1.num > circle2.num ? { ...circle2 } : { ...circle1 };

				biggerCircle.num += smallerCircle.num;
				biggerCircle.color = colors[biggerCircle.num % 7];

				// 충돌 후 위치 조정
				const angle = Math.atan2(dy, dx);
				const sin = Math.sin(angle);
				const cos = Math.cos(angle);

				biggerCircle.x += cos * (circle1.size - distance / 2);
				biggerCircle.y += sin * (circle1.size - distance / 2);
				const speed = Math.sqrt(
					biggerCircle.dx * biggerCircle.dx + biggerCircle.dy * biggerCircle.dy
				);
				biggerCircle.dx = cos * speed;
				biggerCircle.dy = sin * speed;

				// 각 모서리에서 새로운 공 생성
				const newCircles = [
					createNewCircleAtEdge(smallerCircle, 'topLeft', 0),
					createNewCircleAtEdge(smallerCircle, 'topRight', 1),
					createNewCircleAtEdge(smallerCircle, 'bottomLeft', 2),
					createNewCircleAtEdge(smallerCircle, 'bottomRight', 3)
				];

				return [biggerCircle, ...newCircles];
			} else {
				// 다른 색깔의 공들이 충돌한 경우
				const winner = circle1.num > circle2.num ? { ...circle1 } : { ...circle2 };
				const loser = circle1.num > circle2.num ? { ...circle2 } : { ...circle1 };

				winner.num -= loser.num;
				winner.color = colors[winner.num % 7];

				// 충돌 후 위치 조정
				const angle = Math.atan2(dy, dx);
				const sin = Math.sin(angle);
				const cos = Math.cos(angle);

				winner.x += cos * (circle1.size - distance / 2);
				winner.y += sin * (circle1.size - distance / 2);
				const speed = Math.sqrt(winner.dx * winner.dx + winner.dy * winner.dy);
				winner.dx = cos * speed;
				winner.dy = sin * speed;

				return [winner];
			}
		}
		return [circle1, circle2];
	};

	const createNewCircleAtEdge = (
		baseCircle: CircleData,
		corner: 'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight',
		addNum: number
	): CircleData => {
		const { size } = baseCircle;
		let x, y, dx, dy;

		switch (corner) {
			case 'topLeft':
				x = size / 2;
				y = size / 2;
				dx = Math.abs((Math.random() + 0.5) * 2);
				dy = Math.abs((Math.random() + 0.5) * 2);
				break;
			case 'topRight':
				x = containerWidth - size / 2;
				y = size / 2;
				dx = -Math.abs((Math.random() + 0.5) * 2);
				dy = Math.abs((Math.random() + 0.5) * 2);
				break;
			case 'bottomLeft':
				x = size / 2;
				y = containerHeight - size / 2;
				dx = Math.abs((Math.random() + 0.5) * 2);
				dy = -Math.abs((Math.random() + 0.5) * 2);
				break;
			case 'bottomRight':
				x = containerWidth - size / 2;
				y = containerHeight - size / 2;
				dx = -Math.abs((Math.random() + 0.5) * 2);
				dy = -Math.abs((Math.random() + 0.5) * 2);
				break;
		}

		const newNum = baseCircle.num + addNum;

		return {
			...baseCircle,
			id: getNextId(),
			num: newNum,
			color: colors[newNum % 7],
			x,
			y,
			dx,
			dy,
			ignoreCollision: 30 // 30프레임 동안 충돌 무시
		};
	};

	const animate = () => {
		if (!gameRunning) return;

		let newCircles: CircleData[] = [];

		circles.forEach((circle) => {
			let { x, y, dx, dy, size, ignoreCollision } = circle;

			// 다음 위치 계산
			let nextX = x + dx;
			let nextY = y + dy;

			// 벽과의 충돌 처리
			if (nextX > containerWidth - size / 2) {
				nextX = containerWidth - size / 2;
				dx = -dx;
			} else if (nextX < size / 2) {
				nextX = size / 2;
				dx = -dx;
			}

			if (nextY < size / 2) {
				nextY = size / 2;
				dy = -dy;
			} else if (nextY > containerHeight - size / 2) {
				nextY = containerHeight - size / 2;
				dy = -dy;
			}

			// 충돌 무시 카운터 감소
			if (ignoreCollision > 0) {
				ignoreCollision--;
			}

			newCircles.push({ ...circle, x: nextX, y: nextY, dx, dy, ignoreCollision });
		});

		// 공끼리의 충돌 처리
		for (let i = 0; i < newCircles.length; i++) {
			for (let j = i + 1; j < newCircles.length; j++) {
				const result = checkCollision(newCircles[i], newCircles[j]);
				if (result.length !== 2) {
					newCircles.splice(i, 1);
					newCircles.splice(j - 1, 1);
					newCircles.push(...result);
					i--;
					break;
				} else if (result[0] !== newCircles[i] || result[1] !== newCircles[j]) {
					// 충돌 결과가 변경되었을 경우
					newCircles[i] = result[0];
					newCircles[j] = result[1];
				}
			}
		}

		circles = newCircles;

		if (circles.length <= 1) {
			stopGame();
		} else {
			animationId = requestAnimationFrame(animate);
		}
	};
</script>

<div class="game-container">
	<div class="canvas-container" bind:this={container}>
		{#each circles as circle (circle.id)}
			<div
				class="circle-wrapper"
				style="left: {circle.x - circle.size / 2}px; top: {circle.y -
					circle.size / 2}px; width: {circle.size}px; height: {circle.size}px;"
			>
				<Circle num={circle.num} size={circle.size} color={circle.color} />
			</div>
		{/each}
		{#if gameState === 'running'}
			<div class="timer">Time: {timer} seconds</div>
		{/if}
	</div>
	<div class="button-container">
		<button class="start-button" on:click={startGame}>Start</button>
		<button class="reset-button" on:click={resetGame}>Reset</button>
	</div>
	<div class="rules">
		<h3>게임 규칙:</h3>
		<ol>
			<li>Start 버튼을 눌러 게임을 시작합니다.</li>
			<li>숫자가 적힌 공들이 화면에 나타나 움직입니다.</li>
			<li>같은 색의 공들이 충돌하면 큰 숫자의 공이 작은 숫자를 흡수합니다.</li>
			<li>흡수된 공의 숫자만큼 4개의 새로운 공이 각 모서리에서 생성됩니다.</li>
			<li>
				다른 색의 공들이 충돌하면 큰 숫자의 공이 작은 숫자만큼 감소하며 작은 숫자의 공은 사라집니다.
			</li>
			<li>숫자가 같은 다른 색 공들이 충돌하면 둘 다 사라집니다.</li>
			<li>마지막 1개의 공이 남을 때까지 게임이 계속됩니다.</li>
			<li>Reset 버튼을 눌러 언제든지 게임을 초기화할 수 있습니다.</li>
		</ol>
	</div>
</div>

<style>
	.game-container {
		display: flex;
		flex-direction: column;
		align-items: center;
		margin: 50px auto;
		max-width: 800px;
	}

	@media (max-width: 768px) {
		.game-container {
			margin: 10px 10px 100px 10px;
			width: calc(100% - 20px);
		}
	}

	.canvas-container {
		position: relative;
		width: 100%;
		height: 600px;
		max-height: 100vh;
		overflow: hidden;
		box-shadow: 0 0 20px rgba(38, 250, 119, 0.5);
		border-radius: 10px;
	}

	.circle-wrapper {
		position: absolute;
		transition:
			left 0.1s linear,
			top 0.1s linear;
	}

	.timer {
		position: absolute;
		bottom: 10px;
		left: 50%;
		transform: translateX(-50%);
		background-color: rgba(0, 0, 0, 0.5);
		color: white;
		padding: 5px 10px;
		border-radius: 5px;
		font-size: 14px;
	}

	.button-container {
		display: flex;
		justify-content: space-between;
		width: 100%;
		margin-top: 20px;
		z-index: 500;
	}

	.start-button,
	.reset-button {
		padding: 10px 20px;
		font-size: 16px;
		color: white;
		border: none;
		border-radius: 5px;
		cursor: pointer;
		transition: background-color 0.3s;
	}

	.start-button {
		background-color: #4caf50;
		margin: 0 auto;
	}

	.reset-button {
		background-color: #f44336;
	}

	.start-button:hover {
		background-color: #45a049;
	}

	.reset-button:hover {
		background-color: #d32f2f;
	}

	.rules {
		margin-top: 20px;
		text-align: left;
		max-width: 600px;
		padding: 15px;
		border-radius: 10px;
		box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
		color: #999;
	}

	.rules h3 {
		margin-top: 0;
	}

	.rules ol {
		padding-left: 20px;
	}

	.rules li {
		margin-bottom: 10px;
	}
</style>
