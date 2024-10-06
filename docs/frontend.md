---
title: The frontend
nav_order: 2
---

# The frontend

The frontend has two simple goals—fetch ASSIST data from AWS and render that data. ASSIST.org has CORS enabled, which means that their JSON agreements can only be fetched from a backend like AWS Lambda.

![image](https://github.com/user-attachments/assets/72be3a54-c294-40fd-843a-9e9968d9dc35)

The complexity lies in the way that articulation data is fetched. In the case where certain articulations don't already exist in DynamoDB, a lambda function must fetch and cache articulations for future use. In order for the lambda function to do this, it must receive an array of agreement links from the frontend.

There are 116 California Community Colleges.

For more information and reasoning regarding this choice, please view the [backend] documentation.

---

[TransferVision]: https://github.com/michaelrgarcia/transfer-vision
[ASSIST.org]: https://assist.org/
