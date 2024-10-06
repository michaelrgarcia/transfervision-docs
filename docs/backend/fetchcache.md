---
title: Fetch-cacher
nav_order: 1
parent: The backend
---

# Fetch-cacher

Before getting into this Lambda function's logic, a fundamental question needs to be addressed.

> Why aren't more ASSIST.org articulations cached in production?

An operation like this is very slow and costly. This question also misinterprets TransferVision's goal of serving articulations for a *single class*. Most of this data would go unused, since certain classes may have a large amount of articulations across the California Community College system. I've found that streaming articulation data from this Lambda function while caching it is the best course of action.

With that aside, here is a diagram depicting this Lambda function's workflow. This happens 4 times in a single articulation fetching / caching job.

![72be3a54-c294-40fd-843a-9e9968d9dc35](https://github.com/user-attachments/assets/26025cf8-7e0e-4ad4-bdc6-22f28e347ed3)

As it seems, this diagram is the same one shown in the frontend section of this documentation. The frontend plays a role of equal weight in this operation, as it sends the Lambda function chunks of links to make requests with and **finalizes** the caching job.

The "finalization" of the caching job will be addressed shortly. Let's first address each step prior to this.

## Finding a match

```js
export function getMatch(articulationData, courseId, agreementLink) {
  let match;

  if (articulationData && articulationData.articulations && agreementLink) {
    const availableArticulations = deNest(articulationData.articulations);

    for (let j = 0; j < availableArticulations.length; ) {
      const articulationObj = availableArticulations[j].articulation;
      const receivingCourse = getReceivingCourse(articulationObj);
      const articulationId = getId(receivingCourse);

      if (parseInt(articulationId) === parseInt(courseId)) {
        const ccName = getCollegeName(articulationData);
        const articulations = getSendingCourses(articulationObj, ccName);

        if (articulations && Array.isArray(articulations)) {
          articulations.push({ agreementLink });
          match = articulations;
        }
      }

      j += 1;
    }
  }
  

  return match;
}
```

All courses and articulations in an ASSIST.org JSON agreement have a ```courseIdentifierParentId```. When lower division courses are rendered on the frontend, they have access to their ```courseIdentifierParentId``` through a data attribute. *That* is where the ```courseId``` parameter of ```getMatch()``` comes from.

Each articulation's ```courseIdentifierParentId``` is then compared with the given ```courseId``` and pushes matching articulation data into the final ```match``` array.

## Main functionality

The "matching" functionality is the crux of the main generator function below.

```js
async function* getChunkJson(links, courseId, year) {
  const linksArray = Object.values(links); // array of links

  for (const obj of linksArray) {
    const link = obj.link;
    const agreementLink = obj.agreementLink;

    try {
      const response = await fetch(link);
      const json = await response.json(); // turns JSON into obj
      const articulationData = Object.values(json)[0];

      const match = getMatch(articulationData, courseId, agreementLink);

      if (match && match !== null) {
        yield JSON.stringify({ result: match }) + "\n";
        
        await cacheArticulation({ result: match }, `${courseId}_${year}`);
      } else {
        yield JSON.stringify({ error: "No articulation." }) + "\n";
      }
    } catch (error) {
      console.error(`error getting articulations: ${error}`);
    }
  }
}
```

Once a match is found, the generator is streamed through functionality derived from this [documentation entry](https://nodejs.org/api/stream.html#streams-compatibility-with-async-generators-and-async-iterators). The articulation data is then cached by its **fullCourseId**. This is a value consisting of the ```courseId``` and ```academicYear``` chosen on the frontend. 

## Finalization

There is special "finalization" logic as a result of [cost-saving methods]. It's supposed to be more of a trade-off, but also addresses the case in which ```getArticulationData()``` on the frontend is somehow disrupted.

![image](https://github.com/user-attachments/assets/108d0c7e-9ed2-4699-b28a-fddd95aeb2d3)

As shown in the diagram, the connection between the frontend and the Lambda function is disrupted while articulation data is being cached. This creates the possibility of a user obtaining incomplete articulation data.

To address this, a separate Lambda function invoked by API Gateway changes the articulation data's "cacheStatus" DynamoDB attribute from "pending" to "complete" at the end of ```getArticulationData()```. 

![image](https://github.com/user-attachments/assets/e6b75e16-cba1-4a17-8bad-d9a69f70c7fc)

Any DynamoDB queries for articulations with cacheStatus "pending" will fail until cacheStatus is "complete" or until the incomplete data is deleted after 2 miuntes. This is a simpler approach than attempting to continue where ```getArticulationData()``` left off.

----
