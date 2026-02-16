# Server Actions Patterns

## Form with Validation and Error Handling

```tsx
// actions/user.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

const updateProfileSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  bio: z.string().max(500, 'Bio must be under 500 characters').optional(),
});

type ActionState = {
  success?: boolean;
  error?: string;
  fieldErrors?: Record<string, string[]>;
};

export async function updateProfile(
  prevState: ActionState,
  formData: FormData
): Promise<ActionState> {
  const parsed = updateProfileSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    bio: formData.get('bio'),
  });

  if (!parsed.success) {
    return { fieldErrors: parsed.error.flatten().fieldErrors };
  }

  try {
    await db.user.update({
      where: { id: getCurrentUserId() },
      data: parsed.data,
    });
  } catch (error) {
    if (isUniqueConstraintError(error)) {
      return { error: 'Email already in use' };
    }
    return { error: 'Failed to update profile' };
  }

  revalidatePath('/settings');
  return { success: true };
}

// components/profile-form.tsx
'use client';
import { useActionState } from 'react';
import { updateProfile } from '@/actions/user';

export function ProfileForm({ user }: { user: User }) {
  const [state, action, pending] = useActionState(updateProfile, {});

  return (
    <form action={action}>
      {state.error && <div className="text-red-500">{state.error}</div>}
      {state.success && <div className="text-green-500">Profile updated!</div>}

      <label>
        Name
        <input name="name" defaultValue={user.name} required />
        {state.fieldErrors?.name && (
          <span className="text-red-500">{state.fieldErrors.name[0]}</span>
        )}
      </label>

      <label>
        Email
        <input name="email" type="email" defaultValue={user.email} required />
        {state.fieldErrors?.email && (
          <span className="text-red-500">{state.fieldErrors.email[0]}</span>
        )}
      </label>

      <label>
        Bio
        <textarea name="bio" defaultValue={user.bio || ''} />
      </label>

      <SubmitButton />
    </form>
  );
}

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Saving...' : 'Save Changes'}
    </button>
  );
}
```

## File Upload with Server Action

```tsx
// actions/upload.ts
'use server';

import { writeFile } from 'fs/promises';
import { join } from 'path';

const MAX_SIZE = 5 * 1024 * 1024; // 5MB
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];

export async function uploadAvatar(formData: FormData) {
  const file = formData.get('avatar') as File;

  if (!file || file.size === 0) {
    return { error: 'No file selected' };
  }

  if (file.size > MAX_SIZE) {
    return { error: 'File must be under 5MB' };
  }

  if (!ALLOWED_TYPES.includes(file.type)) {
    return { error: 'Only JPEG, PNG, and WebP allowed' };
  }

  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);

  const filename = `${crypto.randomUUID()}.${file.type.split('/')[1]}`;
  const path = join(process.cwd(), 'public', 'uploads', filename);
  await writeFile(path, buffer);

  await db.user.update({
    where: { id: getCurrentUserId() },
    data: { avatar: `/uploads/${filename}` },
  });

  revalidatePath('/settings');
  return { success: true, url: `/uploads/${filename}` };
}
```

## Optimistic Updates Pattern

```tsx
'use client';
import { useOptimistic, useTransition } from 'react';
import { toggleLike } from '@/actions/post';

export function LikeButton({ postId, liked, count }: {
  postId: string;
  liked: boolean;
  count: number;
}) {
  const [optimistic, setOptimistic] = useOptimistic(
    { liked, count },
    (state, newLiked: boolean) => ({
      liked: newLiked,
      count: state.count + (newLiked ? 1 : -1),
    })
  );
  const [isPending, startTransition] = useTransition();

  return (
    <button
      onClick={() => {
        startTransition(async () => {
          setOptimistic(!optimistic.liked);
          await toggleLike(postId);
        });
      }}
      disabled={isPending}
    >
      {optimistic.liked ? '❤️' : '🤍'} {optimistic.count}
    </button>
  );
}
```

## Progressive Enhancement

```tsx
// Forms that work WITHOUT JavaScript
// Server Action processes form submission natively

// app/contact/page.tsx (Server Component)
import { submitContact } from '@/actions/contact';

export default function ContactPage() {
  return (
    <form action={submitContact}>
      <input name="name" required />
      <input name="email" type="email" required />
      <textarea name="message" required />
      <button type="submit">Send</button>
      {/* Works even with JS disabled — standard form POST */}
    </form>
  );
}

// Enhance with client-side features
// components/enhanced-contact-form.tsx
'use client';
export function EnhancedContactForm() {
  const [state, action, pending] = useActionState(submitContact, null);

  return (
    <form action={action}>
      {/* Same form, but now with pending states and inline errors */}
      <input name="name" required />
      <button disabled={pending}>
        {pending ? 'Sending...' : 'Send'}
      </button>
    </form>
  );
}
```

## Delete with Confirmation

```tsx
// actions/todo.ts
'use server';
export async function deleteTodo(id: string) {
  await db.todo.delete({ where: { id } });
  revalidatePath('/todos');
}

// components/delete-button.tsx
'use client';
export function DeleteButton({ id }: { id: string }) {
  const [isPending, startTransition] = useTransition();

  return (
    <button
      disabled={isPending}
      onClick={() => {
        if (!confirm('Are you sure?')) return;
        startTransition(() => deleteTodo(id));
      }}
    >
      {isPending ? 'Deleting...' : 'Delete'}
    </button>
  );
}
```
