---
title: Validation
---

Người dùng rất tinh nghịch, họ sẽ gửi mọi loại dữ liệu vô nghĩa nếu có cơ hội. Để ngăn họ gây ra hỗn loạn, việc kiểm tra tính hợp lệ của dữ liệu biểu mẫu là rất quan trọng.

Lớp kiểm tra đầu tiên là [built-in form validation _(kiểm tra tích hợp trong form)_](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation#using_built-in_form_validation) của trình duyệt, ví dụ đánh dấu một `<input>` là `required`:

```svelte
/// file: src/routes/+page.svelte
<form method="POST" action="?/create">
	<label>
		add a todo
		<input
			name="description"
			autocomplete="off"
			+++required+++
		/>
	</label>
</form>
```

Hãy thử nhấn Enter khi <input> trống.


Loại kiểm tra này là hữu ích, nhưng không đủ. Một số quy tắc kiểm tra (ví dụ: tính duy nhất) không thể thực hiện được bằng các thuộc tính của <input>. Trong một vài trường hợp, nếu người dùng là một hacker giỏi, họ có thể xóa các thuộc tính bằng cách sử dụng công cụ phát triển của trình duyệt. Để đề phòng những hành vi này, bạn luôn nên kiểm tra tính hợp lệ từ phía server _(server-side validation)_. 

Trong `src/lib/server/database.js`, hãy kiểm tra xem mô tả có tồn tại và là duy nhất không:

```js
/// file: src/lib/server/database.js
export function createTodo(userid, description) {
+++	if (description === '') {
		throw new Error('todo cần có mô tả');
	}+++

	const todos = db.get(userid);

+++	if (todos.find((todo) => todo.description === description)) {
		throw new Error('todo phải là duy nhất');
	}+++

	todos.push({
		id: crypto.randomUUID(),
		description,
		done: false
	});
}
```

Hãy thử gửi một todo trùng lặp. SvelteKit đưa chúng ta đến một trang lỗi trông không thân thiện cho lắm. Trên server, chúng ta thấy lỗi 'todo phải là duy nhất', nhưng SvelteKit giấu những thông báo lỗi không mong muốn khỏi người dùng vì chúng thường chứa dữ liệu nhạy cảm.

Tốt hơn là ta giữ người dùng trên cùng một trang, và cung cấp thêm thông tin về lỗi giúp người dùng có thể tự sửa được. Để làm điều này, ta có thể dùng hàm `fail` để trả về dữ liệu từ action cùng với mã trạng thái HTTP phù hợp:

```js
/// file: src/routes/+page.server.js
+++import { fail } from '@sveltejs/kit';+++
import * as db from '$lib/server/database.js';

export function load({ cookies }) {...}

export const actions = {
	create: async ({ cookies, request }) => {
		const data = await request.formData();

+++		try {+++
			db.createTodo(cookies.get('userid'), data.get('description'));
+++		} catch (error) {
			return fail(422, {
				description: data.get('description'),
				error: error.message
			});
		}+++
	}
```

Trong `src/routes/+page.svelte`, chúng ta có thể truy cập giá trị được trả về qua prop `form` (chỉ tồn tại sau khi form đã được gửi):

```svelte
/// file: src/routes/+page.svelte
<script>
	export let data;
	+++export let form;+++
</script>

<h1>todos</h1>

+++{#if form?.error}
	<p class="error">{form.error}</p>
{/if}+++

<form method="POST" action="?/create">
	<label>
		thêm a todo:
		<input
			name="description"
			+++value={form?.description ?? ''}+++
			autocomplete="off"
			required
		/>
	</label>
</form>
```

> Bạn cũng có thể trả về dữ liệu từ một action _mà không cần_ bao nó trong `fail`, ví dụ như để hiển thị một thông báo 'thành công!' khi đã lưu trữ thành công. Ta truy cập dữ liệu này thông qua prop `form`.
