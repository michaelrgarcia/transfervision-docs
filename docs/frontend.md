---
title: The frontend
nav_order: 2
---

# The frontend

The frontend has two simple goals—fetch ASSIST data from AWS and render that data. ASSIST.org has CORS enabled, which means that their JSON agreements can only be fetched from a backend like AWS Lambda.

![image](https://github.com/user-attachments/assets/72be3a54-c294-40fd-843a-9e9968d9dc35)

Most of the logic that needs explaining is the logic for [requesting articulations].

move below section to new md file

## Requesting articulations

The complexity lies in the way that articulation data is fetched. In the case where certain articulations don't already exist in DynamoDB, a lambda function must fetch and cache articulations for future use. In order for the lambda function to do this, it must receive an array of ASSIST API endpoint links from the frontend.

```js
// Creates the array of ASSIST API endpoint links and agreement links

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

This function returns an array of 116 objects containing ASSIST API endpoint links and agreement links. This array is then passed into the function that controls the fetch / rendering operation.



For more information and reasoning regarding this choice, please view the [backend] documentation.

---

[TransferVision]: https://github.com/michaelrgarcia/transfer-vision
[ASSIST.org]: https://assist.org/
