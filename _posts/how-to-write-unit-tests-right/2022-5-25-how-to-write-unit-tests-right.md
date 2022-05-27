---
layout: post
title: How to write unit tests in the right way
tags: [test, unit-test]
date: 2022-5-25
categories: test
---

# tl;dr
1. Code that can be tested easily is good code
   1. If your unit test looks complicated, it's because of your code.
2. Do not test dependencies in unit tests
   1. In unit tests, write tests only for your own code in the unit.
   2. Some code that you didn't write in the unit, 3rd-party API or native API, are all dependencies.
   3. You assume your dependencies are already battle-tested. Period.
   4. As soon as you test dependencies, it becomes integration testing.

# Well... I'm a dev. Do I need to write unit tests...?
Yes. There are different types of tests, such as unit test, integration test, E2E test, and so on.
Developers are definitely held account for writing unit tests.
I see many developers complain that unit tests don't catch all bugs and downplay the importance of it.

However, there is not a single test mechanism that can catch all bugs in a program.
Tests should be built in layers. And the unit test is the bare minimum,
developers need to put unit tests for the code quality. With the stacked layers of unit test,
integration test, and E2E, we decrease the probability of bugs happening.

And just like coding skill, you'll need to build testing skill set in order to quickly
and concisely write unit tests. Let me share the principles that I stick to and how those
helped my unit testing.

# Principle 1: Code that can be tested easily is good code.
Let's first look at an example. In the below example, you have a function that takes
`userId` as an input, and make API calls to `/api/v2/user` and `/api/v2/comment`.
And there you also need to write some `if/else` logics to do sanity check.

You can see that in order to cover all business logics, we would need at least seven test cases.
But let's analyze the code whether this is good code or not.

First, the API calls can be encapsulated for reusability. What if other code also need
functionality to call such APIs? Then, you might end up making the API calls again,
and have sanity checks again in the other code.

By modularzing the API calls, your code will be a lot simpler.

```js
// Other code can possibly similar logic - code is not reusable
const fetchCommentsByUserId = (userId) => {
  const userResponse = fetch("/api/v2/user", {
    userId,
  });
  const userJson = userResponse.json();
  if (!userJson || userJson.body || userJson.body.posts) {
    return [];
  }
  const postsIds = userJson.body.posts.map(post => post.id)
  const commentResponse = fetch("/api/v2/comment", {
    postIds,
  });
  const commentJson = commentResponse.json();
  if (!commentsJson || !commentsJson.body || !commentsJson.comments) {
    return [];
  }
  return commentJson.body.comments;
};

describe("fetchCommentByUserId", () => {
  it("returns empty array if userJson is empty");
  it("returns empty array if userJson.body is empty");
  it("returns empty array if userJson.body.posts is empty");
  it("returns empty array if commentJson is empty");
  it("returns empty array if commentJson.body is empty");
  it("returns empty array if commentJson.body.comments is empty");
  it("returns comments from the comment API otherwise");
});
```

The next example is after we've encapsulated API call logics and sanity checks in modules.
And now, your function's business logics became ridiculously simple. Just pass the `userId`
to `userRepo.findPosts`, and pass the return value of `userRepo.findPosts` to
`postRepo.findComments`. And the output of the function is the return value of `postRepo.findComments`.

As your code got simpler, the unit tests got a lot simpler as well.
You've noticed that I've mocked `userRepo` and `postRepo` in the function's unit test.
That's because now those two repos are dependencies of your function.

```js
/**
 * (1) userRepo and postRepo encapsulate the service logics and reusable
 * (2) userRepo and postRepo handles edge cases on their own.
 */
const fetchCommentsByUserIdRefactored = (userId) => {
  const postIds = userRepo.findPosts(userId).map(post => post.id);
  return postRepo.findComments(postIds);
};

describe("fetchCommentsByUserIdRefactored", () => {
  const spyOnFindPosts = jest.spyOn(userRepo, "findPosts");
  const spyOnFindComments = jest.spyOn(postRepo, "findComments");
  it("calls userRepo and postRepo and return the response of findComments", () => {
    const returnValue = "hello-i-am-return-value";
    spyOnFindComments.mockReturn(returnValue);
    const result = fetchCommentsByUserIdRefactored(1);
    expect(result).toBe(returnValue);
    expect(spyOnFindPosts).toHaveBeenCalled();
    expect(spyOnFindComments).toHaveBeenCalled();
  });
});
```

You may ask since we need to write unit tests for `userRepo` and `postRepo`,
there is no difference between the total amount of unit tests that we need to write.
That's half true and half false.

Of course, we need to unit test both two new modules. However, this encapsulation will
make sure each unit will be tested only once. In the first example, if other code also uses API calls,
you would end up having test cases covering the same business logics.
However, in the second example, you write tests for both modules, and whenever those are called
in other functions, you just mock them.

With this way, it'll reduce the total amount of test cases you'd write.
And you'll benefit from this in the long run.

# Principle 2: Do not test dependencies in your test
To make your unit testing efficient and concise,
you should not test dependencies, but only the code you wrote in the unit.
And that's where mocking comes in. And I've already done mocking in the previous section.

But what I repeatedly observe is that people think mocking makes unit tests fake and useless.
Truly, it's not.
Mocking is a core part of whole unit tests. If you don't allow mocking,
then your tests are no more unit tests, but integration tests.

If you don't mock dependencies, what can normally happen in web app is that your unit tests
make real API calls. If it does, then it'll not only add overhead to your unit tests
but also make your tests flaky. You'll end up either waiting for unit tests that
take more than five minutes or rerun CI/CD builds because your unit tests failed due
to network connection.
We should avoid this situation in all cases.

Let's take a look at another example. The example is a pseudocode that shows
the dependency tree of `helloWorld` function.

If you don't mock dependencies, then you'd need to go through all internal logics of dependencies.
And it'll make your unit tests ridiculously complicated. And what if other function also uses
those dependencies?
Then, in most of your unit tests, you spend a majority of time to test dependencies,
not the code that you've introduced in the function.

```
helloWorld
  - fooBoo
  - yellowMonkey
    - canBeTrue
    - eatsBanana
      - checkIfRotten
        - updateMonkeyHungriness
            - setState
  - numBlueDog
  - resuilt = 1 + 2 + return value of bluedog
  - return result
```

Now, with that said, let's write unit tests for the function.
No matter how complicated the `yellowMonkey` side effect function, you just don't care,
and simply mock it.

Here, you use an assumption that your dependencies are already battle-tested.
Hence, for the unit tests of `helloWorld`, you simply mock the dependencies, and focus on
the new code in the function.

```js
const helloWorld = () => {
  fooBoo();
  yellowMonkey();
  const num = numBlueDog();
  var result = 1 + 2 + num;
  return result
}

// helloWorld.js
describe("helloWorld", () => {
  it("returns 3 + the return value of numBlueDog", () => {
    const returnValue = 5;
    numBlueDog.mockReturn(returnValue);
    const result = helloWorld();
    fooBoo.toHaveBeenCalled();
    yellowMonkey.toHaveBeenCalled();
    blueDog.toHaveBeenCalled();
    expect(result).toBe(3 + returnValue);
  });
});
```

# Final Thoughts
Before having those principles in place, I was one of the devs who doubt about unit tests.
It was because the process to write unit tests seemed to affect my productivity, and seemed
like the ROI of the unit tests is terribly low.

But indeed, unit tests are for your productivity and the ROI of unit tests should be high.
If none of them are true, you're doing something wrong with your unit tests!

Now, with the principles installed in mind, the unit tests got so much easier, and
I like writing unit tests always in my code changes.

And the good thing about unit tests is that it serves as regression tests by default. 
Hence, its value grows significantly over time not only for yourself but also for the entire team.

Hope this sharing also help your unit tests makes easier and something enjoyable in your dev life.
