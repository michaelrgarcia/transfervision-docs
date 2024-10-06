---
title: Requesting articulations
nav_order: 1
parent: The frontend
---

# Requesting articulations

Most of the frontend's complexity lies in the way that articulation data is fetched. In the case where certain articulations don't already exist in DynamoDB, a Lambda function must fetch and cache articulations for future use. In order for the Lambda function to do this, it must receive an array containing ASSIST API endpoint links from the frontend.

```js
// Creates an array of ASSIST API endpoint links and agreement links

async function getArticulationParams(receivingId, majorKey, year) {
  const articulationParams = [];
  const communityColleges = await getCommunityColleges();

  const receiving = receivingId;
  const key = majorKey;

  communityColleges.forEach((college) => {
    if (college.id) {
      const sending = college.id;
      const link = `${process.env.ASSIST_API_PARAMS}=${year}/${sending}/to/${receiving}/Major/${key}`;
      const agreementLink = `${process.env.ASSIST_AGREEMENT_PARAMS}=${year}&institution=${sending}&agreement=${receiving}    &agreementType=to&view=agreement&viewBy=major&viewSendingAgreements=false&viewByKey=${year}/${sending}/to/${receiving}/Major/${key}`;

      articulationParams.push({ link, agreementLink });
    }
  });

  return articulationParams;
}
```

This function returns an array of objects containing ASSIST API endpoint links and agreement links. This array is then passed into the function that controls the fetch / rendering operation.

For more information and reasoning regarding this choice, please view the [backend] documentation.
