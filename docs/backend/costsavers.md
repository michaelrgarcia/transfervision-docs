---
title: Cost-Saving Methods
nav_order: 3
parent: The backend
---

# Cost-Saving Methods

The way Lambda measures cost through [GB-seconds](https://aws.amazon.com/lambda/pricing/) dictates some of TransferVision's design choices.

Lambda is a pay-for-what-you-use service. AWS measures costs through how long a function invocation is active, so I have done my best to shorten execution times in areas where it may be lengthy.

In particular, the process of manually fetching articulations from ASSIST.org has the longest execution time out of all of TransferVision's components. The chunking logic documented in this [frontend entry](https://michaelrgarcia.github.io/transfervision-docs/docs/frontend/articulationFetching.html) combats this by utilizing 4 invocations in rapid succession, rather than letting costs rise as a result of 1 big fetching job.

The completion logic on the frontend (shown below) is what makes this method insusceptible to user error. This code runs when ```getArticulationData()``` is successful.

```js
async function finalizeSearch(fullCourseId, articulations) {
  const cacheFinalizer = process.env.CACHE_COMPLETER;

  if (articulations) {
    organizeArticulations();

    try {
      await fetch(cacheFinalizer, {
        body: JSON.stringify({
          fullCourseId,
        }),
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
      });

      showCidSlider();
    } catch (err) {
      console.error(`error finalizing cache job: ${err}`);
      cacheFinalizeError(); 
    }
  }
}
```

----
