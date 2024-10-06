---
title: Requesting articulations
nav_order: 1
parent: The frontend
---

# Requesting articulations

Most of the frontend's complexity lies in the way that articulation data is fetched. If articulations don't already exist in DynamoDB, the frontend must salvage articulation data from a Lambda function while the function stores the articulations in DynamoDB. The entire process will be described below.

First, an array of endpoint links needs to be created.

```js
async function getArticulationParams(receivingId, majorKey, year) {
  const articulationParams = [];
  const communityColleges = await getCommunityColleges();

  const receiving = receivingId;
  const key = majorKey;

  communityColleges.forEach((college) => {
    if (college.id) {
      const sending = college.id;
      const link = `${process.env.ASSIST_API_PARAMS}=${year}/${sending}/to/${receiving}/Major/${key}`;
      const agreementLink = `${process.env.ASSIST_AGREEMENT_PARAMS}=${year}&institution=${sending}&agreement=${receiving}&agreementType=to&view=agreement&viewBy=major&viewSendingAgreements=false&viewByKey=${year}/${sending}/to/${receiving}/Major/${key}`;

      articulationParams.push({ link, agreementLink });
    }
  });

  return articulationParams;
}
```

This array is then passed into the function that controls the fetch / rendering operation.

```js
export async function getArticulationData(links, courseId, year) {
  const fullCourseId = `${courseId}_${year}`; // Explained in the backend docs.
  const processingQueue = links.slice(); // A shallow copy of "links" is needed for the progress tracker.

  const abortController = new AbortController();
  const { signal, isAborted } = abortHandler(abortController);

  const updateProgress = createProgressTracker(links.length); // The amount of California Community Colleges may differ in the future.

  let articulations;

  window.addEventListener("beforeunload", () => abortController.abort());

  showResults();
  updateProgress(0);

  try {
    articulations = await getClassFromDb(
      fullCourseId,
      links.length,
      updateProgress,
    );

    if (!articulations) {
      articulations = await processChunks(
        processingQueue,
        signal,
        courseId,
        updateProgress,
        year,
      );

      hideLoadingContainer();

      if (!isAborted) {
        await finalizeSearch(fullCourseId, articulations); // Segues into the backend docs.
      }
    }
  } catch (error) {
    if (error.name === "AbortError") {
      console.log("requests aborted due to page unload");
    } else {
      console.error("error processing requests", error);
    }
  } finally {
    window.removeEventListener("beforeunload", () => abortController.abort());
  }

  return { articulations, updateProgress };
}
```

Let's focus on the function ```processChunks()```.

```js
async function processChunks(
  processingQueue,
  signal,
  courseId,
  updateProgress,
  year,
  assistArticulations = [],
) {
  const concurrencyLimit = 29;

  let linksChunk;

  if (processingQueue.length === 0) return assistArticulations;

  if (processingQueue.length < concurrencyLimit) {
    linksChunk = processingQueue.splice(0, processingQueue.length - 1);
  } else {
    linksChunk = processingQueue.splice(0, concurrencyLimit);
  }

  try {
    const streamArticulations = await requestArticulations(
      linksChunk,
      signal,
      courseId,
      year,
      updateProgress,
    );

    assistArticulations.push(...streamArticulations);

    return await processChunks(
      processingQueue,
      signal,
      courseId,
      updateProgress,
      year,
      assistArticulations,
    ); // recursive call
  } catch (error) {
    if (error.name === "AbortError") {
      console.log("aborted request");
    } else {
      console.error("error processing chunk:", error);
    }
  }
}
```

As the name suggests, this function breaks up the ```processingQueue``` array into smaller chunks for the Lambda function. ```requestArticulations()``` then sends a POST request containing the endpoint links to the Lambda function and renders the streamed JSON response. All of this is done recursively until the ```processingQueue``` array is empty.

For more information and reasoning regarding this design choice, please view the [backend](https://michaelrgarcia.github.io/transfervision-docs/docs/backend/) documentation.

----
