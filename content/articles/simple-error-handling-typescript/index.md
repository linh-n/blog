---
title: "A simple error handling method in TypeScript"
date: "2025-04-21"
summary: "When it comes to error handling in TypeScript, throwing errors is not the best approach, especially where errors are expected. Let's explore a better alternative."
description: "Simple yet effective error handling in TypeScript."
tags: ["typescript"]
showTags: true
toc: false
math: false
readTime: false
hideBackToTop: false
---

## Try/catch hell

It's a pain of try catching all possible errors, especially when dealing with network requests. Miss one error? Whole app can crash. Catching too much? Errors go into oblivion and users won't know what went wrong.

## Possible solutions

Many libraries are out there, inspired by Go/Rust style of error handling. They use a pattern of returning an object with either a success or error state. This is great, but it can be cumbersome to implement and requires a lot of boilerplate code.

## A simple method

Instead of using a library, we can create a simple type to handle errors. This type will be a union of the expected result and an error object. This way, we can easily check if the result is an error or not.

### Implementation

```ts
export type ErrorMessage = {
  code?: string;
  traces?: unknown[];
};

type Data<T> = {
  data: T;
  error?: never;
};

type Error = {
  data?: never;
  error: ErrorMessage;
};

export type Result<T> = NonNullable<Data<T> | Error>;
```

The main star here is the `Result` type. Since it's a `NonNullable`, we can be sure that every time, we will have either a `data` or an `error` property. This way, we can easily check if the result is an error or not.

### Usage

Wherever you expect errors to happen, you can use this type. For example, in a network request:

```ts
import { Result } from './types';

export const ERRORS = {
  FETCH_UNAUTHORIZED: 'FETCH_UNAUTHORIZED',
  FETCH_FAILED: 'FETCH_FAILED',
}

export fetchData = async (): Result<ImportantData> => {
  const session = await getSession();
  const token = session?.accessToken;

  if (!token) {
    return { error: { code: ERRORS.FETCH_UNAUTHORIZED } };
  }

  const response = await fetch('https://api.example.com/data', {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });

  const data = await response.json();

  if (!response.ok) {
    return { error: { code: ERRORS.FETCH_FAILED, traces: [data] } };
  }

  return { data as ImportantData };
}
```

And from the presentation layer, we can easily check if the result is an error or not. Let's say we are using React:

```ts
import { useEffect, useState } from "react";
import { fetchData, ERRORS } from "./api";

export const Component = async () => {
  const result = await fetchData();

  if (result.data) {
    return <div>Data: {JSON.stringify(result.data)}</div>;
  }

  if (result.error.code === ERRORS.FETCH_UNAUTHORIZED) {
    return <div>You are not authorized</div>;
  } else {
    return <div>Unexpected error</div>;
  }
};
```
