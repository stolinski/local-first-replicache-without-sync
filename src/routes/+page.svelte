<script lang="ts">
	import './app.css';
	import { dev } from '$app/environment';
	import { Replicache, type WriteTransaction } from 'replicache';
	import { flip } from 'svelte/animate';

	type Todo = {
		id: number;
		name: string;
		deleted: boolean;
		createdAt: string;
		updatedAt: string;
		completed: boolean;
	};

	let todos: Todo[] = $state([]);
	let form_state = $state({ name: '' });
	let name = dev ? 'dev:todos:userId' : 'todos:userId';

	let rep = new Replicache({
		name,
		licenseKey: import.meta.env.VITE_REP_LICENSE,
		mutators: {
			create_todo: async (tx: WriteTransaction, args: Todo) => {
				const key = `todos/${args.id}`;
				await tx.set(key, args);
			},
			toggle_todo: async (tx: WriteTransaction, args: Todo) => {
				const key = `todos/${args.id}`;
				await tx.set(key, args);
			}
		}
	});

	$effect(() => {
		return rep.subscribe(
			async (tx) => {
				const todos_items = await tx.scan({ prefix: 'todos/' }).entries().toArray();
				return todos_items.map(([_, values]) => values as Todo).filter((todo) => !todo.deleted);
			},
			(items: Todo[]) => {
				todos = items
					.sort((a, b) => (a.createdAt < b.createdAt ? 1 : a.createdAt > b.createdAt ? -1 : 0))
					.sort((a, b) => (a.completed === b.completed ? 0 : a.completed ? 1 : -1));
			}
		);
	});

	function toggle(todo: Todo) {
		rep.mutate.toggle_todo({
			...todo,
			completed: !todo.completed
		});
	}

	function onsubmit(e: SubmitEvent) {
		e.preventDefault();
		rep.mutate.create_todo({
			id: new Date().getTime(),
			name: form_state.name,
			deleted: false,
			createdAt: new Date().toISOString(),
			updatedAt: new Date().toISOString(),
			completed: false
		});
		form_state.name = '';
	}
</script>

<form method="POST" {onsubmit}>
	<label for="name">Todo:</label>
	<input type="text" name="name" id="name" bind:value={form_state.name} />
	<button type="submit">+ Add</button>
</form>

<ul>
	{#each todos as todo (todo.id)}
		<li animate:flip={{ duration: 200 }} class:completed={todo.completed}>
			<span>
				{todo.name}
			</span>
			<button class="check" onclick={() => toggle(todo)}>
				<svg xmlns="http://www.w3.org/2000/svg" width="40px" height="100%" viewBox="0 0 32 32"
					><title>c-check</title><g fill="var(--fg)" stroke-linejoin="miter" stroke-linecap="butt"
						><circle
							cx="16"
							cy="16"
							r="14"
							fill="none"
							stroke="var(--fg)"
							stroke-linecap="square"
							stroke-miterlimit="10"
							stroke-width="2"
						></circle><polyline
							points="9 17 13 21.5 23 10"
							fill="none"
							stroke="var(--fg)"
							stroke-linecap="square"
							stroke-miterlimit="10"
							stroke-width="2"
						></polyline></g
					></svg
				>
			</button>
		</li>
	{/each}
</ul>

<style>
	label {
		display: block;
	}

	form {
		margin-bottom: 1rem;
		input,
		button {
			font-size: var(--fs-l);
			background: transparent;
			border: none;
			transition: 0.2s ease background;
			&:hover {
				background: var(--tint-or-shade);
			}
		}
		input {
			border-bottom: 0.5px solid var(--fg);
		}
	}

	ul {
		list-style: none;
		margin: 0;
		padding: 0;
	}

	li {
		display: flex;
		align-items: center;
		gap: 1rem;
		font-size: var(--fs-l);
		& + li {
			margin-top: 0.5rem;
		}
	}

	.completed {
		opacity: 0.3;
		transition: opacity 0.2s ease;
		&:hover {
			opacity: 1;
		}
		span {
			position: relative;
			&:after {
				content: '';
				position: absolute;
				display: block;
				height: 6px;
				background: #333;
				top: 50%;
				left: 0;
				width: 100%;
			}
		}
	}
	.check {
		padding: 0;
		background: transparent;
		border: none;
		display: flex;
		align-items: center;
	}
</style>
