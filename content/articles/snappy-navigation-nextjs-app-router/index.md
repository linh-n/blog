---
title: "Next JS navigation feels slow? Make it snappy again!"
date: "2025-04-23"
lastmod: "2025-04-29"
summary: "No matter how fancy the App router has become with all bells and whistles, the clunky navigation experience always bothers me."
description: "Fix the slow navigation experience in Next.js App router with a simple solution."
tags: ["nextjs", "react", "typescript"]
showTags: true
toc: false
math: false
readTime: false
hideBackToTop: false
---

_TL;DR: go to the [final solution](#final-solution) section to see the final code._

I have been using the App router for a while now, and I have to say that the navigation experience is not as smooth as I would like it to be. I miss the experience of SPA where the page transitions are snappy and instantaneous, when you click on a link, the link itself will instantly change to the _active_ state.

Try to search for something like "next js slow navigation" and you will find a lot of people complaining about the same issue.

## The problem

The problem lies in the fact that the App router relies on server-side rendering (SSR) and static site generation (SSG) to deliver content to the client. Normally the change of pages is quite fast, but as Next JS has to wait for the new page to be rendered on the server, so in unideal situations, depending on how fast the server processes the request, network latency and speed, there is a period of limbo in between two pages.

Moreover, we can't use/watch the navigation hooks (`usePathname`, `useSearchParams` etc.) because the states only update **_after_** the navigation is complete.

## Finding a solution

### Suspense

All these features [`Suspense`](https://react.dev/reference/react/Suspense) and [`loading.js`](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming) are great, but they don't solve the problem of the link itself not being in the active state and current content is already stale. They only help with the loading state of the new page.

### New hook?

Next JS team is very aware of this problem and in the recent release of 15.3, they introduced a new hook called [`useLinkStatus`](https://nextjs.org/blog/next-15-3#uselinkstatus), but I think it is not enough. The hook is designed to provide a way to track the status of a link, but only to the link itself, since it can only be used in the context of a link, so the most you can do is to show some kind of spinner inside the link itself.

### Client state?

Of course we can use client state to track the status of the link with onClick event to get the snappy feeling back, but still there are a few edge cases, like when users use Ctrl/Cmd to open the link in a new tab. Handling these cases are not trivial.

### Hey, new onNavigate event!

In the same release, Next JS team introduced a new event called [`onNavigate`](https://nextjs.org/docs/app/api-reference/components/link#onnavigate), which is fired only when real navigation happens with Link component, and only on client side. This allow us to easily know when the navigation is happening, and we can use this to update the state of the link to show the active state.

One caveat though, setting the state in the `onNavigate` will not happen immediately, because at this point the navigation has already begun and all state updating will be scheduled for later. Are we back to square one? Not really, because we can use another hook introduced in React 18: [`useOptimistic`](https://react.dev/reference/react/useOptimistic) to optimistically update the state of the link before the navigation is complete. Since it's not batched like `useState`, we can show the active state of the link immediately, and then update it again when the navigation is complete.

## Final solution

### 1. The context

Since I want to update other sections on the page to show the loading state as well (fade out the content, show a spinner, etc.), I will make use of react context to share the state of the navigation across the whole app.

Let's create a new context that will hold the state of the navigation and provide a way to update it, and a custom hook to use it more easily.

```tsx
// navigation-context.tsx

"use client";

import { usePathname } from "next/navigation";
import { createContext, ReactNode, useContext, useOptimistic } from "react";

type OptimisticNavigationContextType = {
  isNavigating: boolean;
  optimisticPathname: string;
  setOptimisticPathname: (id: string) => void;
};

type OptimisticNavigationContextProviderProps = {
  children: ReactNode;
};

const OptimisticNavigationContext = createContext<
  OptimisticNavigationContextType | undefined
>(undefined);

/**
 * This context provider will hold the state of the navigation and provide a way to update it.
 * Wrap this around your app to use.
 */
export const OptimisticNavigationContextProvider = ({
  children,
}: OptimisticNavigationContextProviderProps) => {
  const pathname = usePathname();
  const [optimisticPathname, setOptimisticPathname] = useOptimistic(
    pathname,
    (_, action: string) => action
  );

  return (
    <OptimisticNavigationContext.Provider
      value={{
        isNavigating: pathname !== optimisticPathname,
        optimisticPathname,
        setOptimisticPathname,
      }}
    >
      {children}
    </OptimisticNavigationContext.Provider>
  );
};

/**
 * Use this hook to get the state of the navigation and update it from client components.
 */
export const useOptimisticNavigation = () => {
  const context = useContext(OptimisticNavigationContext);
  if (!context) {
    throw new Error(
      "useOptimisticNavigation must be used within a OptimisticNavigationContextProvider"
    );
  }
  return context;
};
```

Then let's wrap the context provider around the app, preferably in the root `layout.tsx`

```tsx
// app/layout.tsx

...

 <OptimisticNavigationContextProvider>
   {children}
 </OptimisticNavigationContextProvider>

...
```

You can keep using server components as usual, since this follows Next JS's [composition patterns](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#using-context-providers).

### 2. The links

Next, in your navigation component, we can use the `useOptimisticNavigation` hook to get the state of the navigation and update it when the `onNavigate` event of a link fires. Make sure to make it a client component, since we are using hooks.

```tsx
// app/components/navigation.tsx
"use client";

...

const { optimisticPathname, setOptimisticPathname } = useOptimisticNavigation();

...

<Link 
  href={path}
  onNavigate={() => setOptimisticPathname(path)}
  className={clsx({ "active": optimisticPathname === path })}
>
  {label}
</Link>
```

If you use `useRouter` to navigate, you can call `setOptimisticPathname` directly.

### 3. The loading state

In your other components that need to show the loading state, you can use the `isNavigating` state to show the loading state. For example, you can use it to fade out the content and show a spinner.

Again, we can use the same pattern, to make a client component and wrap it around any component that needs to show the loading state.

```tsx
// app/components/navigation-wrapper.tsx
"use client";

const NavigationWrapper = ({ children, ...props }: HtmlHTMLAttributes<HTMLDivElement>) => {
  const { isNavigating } = useOptimisticNavigation();

  return <div className={clsx({ "fade-out": isNavigating })} {...props} />;
};

export default NavigationWrapper;
```

## To go further

This implementation relies on `pathname` which assumes that you're not using query params in your links. If this is the case, you have to go extra mile to make sure the `isNavigating` and active link state are updated correctly.

Also, you can add extra logic in the context provider to handle some fancier cases, like to only show the loading state on certain components depending on the link clicked. I'll leave this to you to explore.