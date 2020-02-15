---
title: Two simple tips on using React Hooks
date: '2020-02-15T10:10:43.769Z'
---

I have been using React Hooks for a few months now and I am absolutely loving it! Today I would like to share 2 simple tips that I have learned along the way. Hopefully you will find them useful!

### useEffect for API requests
A common scenario when writing component is to request some data from a REST API and then display them to the users. To do this using hooks, we often do the fetching inside the `useEffect` hook like so:

```js 
useEffect(() => {
	const fetchUserData = async () => {
		const response = await API.getUserData(userId);
		setUserData(response.userData);
	};

	fetchUserData();
}, [userId);
```

This looks fine at first glance but there are actually 2 potential bugs here:
1. If user navigate away before the API returns, this will cause a set state to an unmounted component which may lead to memory leak.
2. If `userId` changes before the previous request returns, this will cause the component to send another request which may lead to race condition (The first request returns slower than the second request and override the state).

To prevent those bugs we can simply add a local variable `ignore`:

```js
useEffect(() => {
	let ignore = false;

	const fetchUserData = async () => {
		const response = await API.getUserData(userId);
		if (!ignore) {
			setUserData(response.userData);
		}
	};

	fetchUserData();
	return () => {
		ignore = true;
	};
}, [userId);
```

`ignore` will be set to true when ever the component is unmounted or the dependency array has changed (`userId` changed in this case). Therefore `setUserData` will not be called!

This is actually documented in [React’s official documentation]([Hooks FAQ – React](https://reactjs.org/docs/hooks-faq.html#is-it-safe-to-omit-functions-from-the-list-of-dependencies)) but I think this is very useful and I want to share this to more people!

### useDeepMemo
Sometimes when we write custom hooks we want that hook to be able to take a dynamic object or array as input. For example consider this `useAPI` custom hook:

```js
// usage
const response = useAPI({
	url: ‘/api/get-user’,
	body: { userId },
});

// implementation
function useAPI({ url, body }) {
	const [response, setResponse] = useState(null);

	useEffect(() => {
		// do the request here
	}, [url, body]);

	return response;
}
```

This is a nice handy custom hook, but there is just one problem. Whenever there is a re-render, `body` is passed in as a new object and `useEffect` will get triggered again. This will cause an infinite loop!

Sure, we can wrap the body object in an `useMemo` hook, but it becomes a pain in the ass when you have to do that every time you use `useAPI`!

To solve this, we can write another custom hook `useDeepMemo`:

```js
// you can also use other deep compare functions
// e.g. lodash’s _.isEqual
import { equal } from ‘@wry/equality’;

function useDeepMemo(memoFn, key) {
  const ref = useRef();

  if (!ref.current || !equal(key, ref.current.key)) {
    ref.current = { key, value: memoFn() };
  }

  return ref.current.value;
}
```

I first saw this technique from [Apollo’s source code](https://github.com/apollographql/apollo-client/blob/master/src/react/hooks/utils/useDeepMemo.ts). This is a replacement for `useMemo` but it uses deep equality to compare memo keys and it guarantees that the memo function will only be called if the keys are unequal.

With that, we can then rewrite our `useAPI` hook:

```js
function useAPI({ url, body }) {
	const [response, setResponse] = useState(null);
	const cachedBody = useDeepMemo(() => body, [body]);

	useEffect(() => {
		// do the request here
	}, [url, cachedBody]);

	return response;
}
```

Now we can use `useAPI` without worrying about accidentally causing an infinite loop!