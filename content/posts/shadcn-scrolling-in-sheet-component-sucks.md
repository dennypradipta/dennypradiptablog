+++
title = "Scrolling in Shadcn's Sheet Component Sucks"
date = 2025-10-30T15:59:48+07:00
images = ["/shadcn-scrolling-in-sheet-component-sucks.webp"]
+++

I keep getting this error in Sentry and it blew through my quota:

![Failed to execute 'contains' on 'Node': parameter 1 is not of type 'Node'.](/shadcn-scrolling-in-sheet-component-sucks-1.png)

Sentry kept pointing at `@radix-ui/react-dialog` via `react-remove-scroll`. I couldn't reproduce it for weeks because I had no idea which component was triggering it. The error stack wasn't helpful either, it's not even pointing to my actual code. So I archived and ignored it in Sentry (yeah, not my proudest moment), and moved on.

A month later I finally found how to reproduce it... by fixing another bug 😂. I was playing with my notification drawer (built with shadcn’s `Sheet`) that has a header, a scrollable content, and a footer.

```jsx
import {
  Sheet,
  SheetContent,
  SheetDescription,
  SheetFooter,
  SheetHeader,
  SheetTitle,
} from '@/components/ui/sheet';

export default function NotificationDrawer({ ...props }) {
  return (
    <Sheet ...>
      <SheetContent
        ....
      >
        <SheetHeader>
          <SheetTitle className='flex w-full items-center justify-between'>
            ...
          </SheetTitle>
          <SheetDescription className='sr-only'>
            ...
          </SheetDescription>
        </SheetHeader>
        {/* This was meant to be scrollable */}
        <div className={cn(`overflow-y-auto text-black`)}>
          {isLoading ? (
            <div className='flex flex-col items-center justify-center gap-2'>
              <Skeleton className='h-32 w-full' />
              <Skeleton className='h-32 w-full' />
              <Skeleton className='h-32 w-full' />
            </div>
          ) : (
            <div>
              ...
            </div>
          )}
        </div>
        <SheetFooter className='text-red-500 p-4'>
          ...
        </SheetFooter>
      </SheetContent>
    </Sheet>
  );
}

```

While scrolling the content, boom — same error. Finally reproducible.

![Finally found it, fuck yeah.](/shadcn-scrolling-in-sheet-component-sucks-2.png)

Root cause: `react-remove-scroll` had a bug. Radix Dialog still depended on the older version. [There’s a PR open to bump it](https://github.com/radix-ui/primitives/issues/3589), but at the time of writing it’s not merged yet.

What worked for me right now is forcing the dependency to `2.7.1` via `overrides` in `package.json`, then reinstall your deps:

```json
// In the package.json
{
  "overrides": {
    "react-remove-scroll": "2.7.1"
  }
}
```

After this, the error stopped showing up on scroll in my Sheet content. Finally. I've won but at what cost? **10.8K fucking Sentry events**.

And remember, if all else fails, there’s always coffee. Lots of coffee.
