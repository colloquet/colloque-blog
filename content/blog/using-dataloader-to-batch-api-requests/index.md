---
title: Using DataLoader to batch API requests
date: '2020-05-23T11:20:52.558Z'
---

### The Problem

Let's say you have a list of user ID as props and you want to fetch and render a list of user's info. You may have an API that looks something like this:

```jsx
// url
const url = '/api/get-users';

// input
const input = {
  userIds: [1, 2, 3],
};

// output
const output = {
  users: [
    // ...list of user object
  ],
};
```

This is great, you pass it a list of user IDs and you can a list of user objects. You can simply do the fetching inside the list component and render the items after getting the list of user objects. This is simple enough, but let's make things more challenging.

What if there is a new component that also needs to fetch a list of users? The list of user ID might be different we cannot abstract the fetching logic because it is at the other side of the React tree.

You can do another fetch in the new component, but this is not ideal because:

- You can potentially save a request by combining the 2 requests
- You might be requesting for the same data twice (some IDs might overlap)

Wouldn't it be great if somehow we can collect all the user IDs that needed to be fetched and combine them into a single request? Well, it turns out you can do just that using [DataLoader](https://github.com/graphql/dataloader)!

### What is DataLoader?

> DataLoader is a generic utility to be used as part of your application's data fetching layer to provide a simplified and consistent API over various remote data sources such as databases or web services via batching and caching.

I came across DataLoader when researching GraphQL. It is used to solve the N + 1 problem in GraphQL, you can learn more about it [here](https://itnext.io/what-is-the-n-1-problem-in-graphql-dd4921cb3c1a). Essentially, it provides APIs for developers to load some keys. All the keys it collects within a single frame of execution (a single tick of the event loop) will be passed into a user-defined batch function.

When using GraphQL, the batching function is usually a call to DB. But when using it in the browser, we can instead define the batching function to send an API request. It will look something like this:

```jsx
import DataLoader from 'dataloader';

async function batchFunction(userIds) {
  const response = await fetch('/api/get-users');
  const json = await response.json();
  const userIdMap = json.users.reduce((rest, user) => ({
    ...rest,
    [user.id]: user,
  }));
  return userIds.map((userId) => userIdMap[userId] || null);
}

const userLoader = new DataLoader(batchFunction);
```

Let's see what's going on here:

- A DataLoader takes in a batch function
- The batch function accepts a list of keys and returns a Promise which resolves to an array of values.
  - The Array of values must be the same length as the Array of keys.
  - Each index in the Array of values must correspond to the same index in the Array of keys.
- The result of our API might not be in the same order as the passed in user IDs and it might skip for any invalid IDs, this is why I am creating a `userIdMap` and iterate over `userIds` to map the value instead of returning `json.users` directly.

You can then use this `userLoader` like this:

```jsx
// get a single user
const user = await userLoader.load(userId);

// get a list of user
const users = await userLoader.loadMany(userIds);
```

You can either use `load` to fetch a single user or `loadMany` to fetch a list of users.

By default, DataLoader will cache the value for each key (`.load()` is a memoized function), this is useful in most cases but in some situations you might want to be able to clear the cache manually. For example if there's something wrong with the user fetching API and the loader is returning nothing for some keys, you probably don't want to cache that. You can then do something like this to clear the cache manually:

```jsx
// get a single user
const user = await userLoader.load(userId);
if (user === null) {
  userLoader.clear(userId);
}

// get a list of user
const users = await userLoader.loadMany(userIds);
userIds.forEach((userId, index) => {
  if (users[index] === null) {
    userLoader.clear(userId);
  }
});
```

With the power of React Hook, you can abstract this user fetching logic into a custom hook:

```jsx
// useUser.js
import { useState, useEffect } from 'react';

import userLoader from './userLoader';

function useUser(userId) {
  const [isLoading, setIsLoading] = useState(false);
  const [user, setUser] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      setIsLoading(true);
      const user = await userLoader.load(userId);
      if (user === null) {
        userLoader.clear(userId);
      }
      setUser(user);
      setIsLoading(false);
    };

    fetchUser();
  }, [userId]);

  return {
    isLoading,
    user,
  };
}

export default useUser;

// use it anywhere in the application
const user = useUser(userId);
```

Isn't this great? Simply use `useUser` in a component and it will take care of the rest for you! You don't need to worry about abstracting the fetching logic or caching the response anymore!

Here is a quick demo:
<iframe
  src="https://codesandbox.io/embed/dataloader-example-t5l1y?fontsize=14&hidenavigation=1&theme=dark"
  style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
  title="DataLoader example"
  allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
  sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
></iframe>

### But what if the components do not render in a single frame?

Worries not, DataLoader allows providing a custom batch scheduler to account for this. As an example, here is a batch scheduler which collects all requests over a 100ms window of time (and as a consequence, adds 100ms of latency):

```jsx
const userLoader = new DataLoader(batchFunction, {
  batchScheduleFn: (callback) => setTimeout(callback, 100),
});
```

### Ok, it looks pretty good so far, is there any downside by using DataLoader?

From my experience, there is one tiny thing that bothers me when using DataLoader. Because DataLoader requires a single frame to collect all keys, it will take at least 2 frames to returns the results, even when it's cached. Meaning if you have a loading indicator, it will still flash for a split second. I have yet to find a solution to this but I will update this post as soon as I find one.

### Conclusion

By using DataLoader, you can easily batch requests initiated from different components anywhere in the render tree, and the result will be cached automatically, you also have the power to customize the scheduler and caching behavior. I have used React Hook as an example but you can easily use it in any other framework as well. What do you think of this pattern? Is there any other pitfalls that I haven't considered? Let me know!

