---
title: Getting C-IDs
nav_order: 2
parent: The backend
---

# Getting C-IDs

Before discussing C-IDs and the challenges they present, I would like to thank [Avelia](https://github.com/avelyssaea) for her contribution to this feature. She wrote the code that downloads a csv file containing all C-IDs by scraping the official [C-ID website](https://c-id.net/).

Here is the code (slightly modified by me):

```js
async function scrapeCids() {
  /**
   * Script Name: scraper.js
   * Description: This script initializes a session to capture cookies and then fetches C-ID data from a specified URL using axios.
   * Author: avelyssaea
   * Date Created: September 7, 2024
   * Version: 1.0
   *
   * Dependencies:
   * - axios: For handling HTTP requests and managing session cookies.
   * - axios-cookiejar-support: For cookie management.
   * - tough-cookie: For handling cookies.
   *
   * Usage:
   * 1. Ensure the required packages are installed (use `npm install axios axios-cookiejar-support tough-cookie puppeteer`).
   * 2. Run the script using Node.js to fetch and save course data to 'courses_data.csv'.
   */

  // Define URLs for requests
  const initialUrl = "https://c-id.net/";
  const coursesUrl = "https://c-id.net/courses";
  const coursesCsvUrl = "https://c-id.net/courses/csv";

  // Create a new CookieJar to store cookies
  const cookieJar = new CookieJar();

  // Create an axios instance with cookie support
  const client = wrapper(
    axios.create({
      jar: cookieJar,
      withCredentials: true,
      headers: {
        accept:
          "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7",
        "accept-language": "en-US,en;q=0.9",
        dnt: "1",
        priority: "u=0, i",
        "sec-ch-ua": '"Not;A=Brand";v="24", "Chromium";v="128"',
        "sec-ch-ua-mobile": "?0",
        "sec-ch-ua-platform": '"macOS"',
        "sec-fetch-dest": "document",
        "sec-fetch-mode": "navigate",
        "sec-fetch-site": "none",
        "sec-fetch-user": "?1",
        "upgrade-insecure-requests": "1",
        "user-agent":
          "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.0.0 Safari/537.36",
      },
    })
  );

  try {
    // Perform the initial request to capture cookies
    await client.get(initialUrl);
    console.log("Session initialized and cookies captured.");

    // Use captured cookies to access the courses page
    await client.get(coursesUrl);
    console.log("Courses page accessed.");

    // Make the request to download the CSV file
    const response = await client.get(coursesCsvUrl, {
      responseType: "arraybuffer",
    });

    return response.data;
  } catch (error) {
    console.error(`An error occurred: ${error}`);

    return false;
  }
}
```

To put it simply, a C-ID makes it easy to identify equivalent courses across the California Community College system. 

C-IDs are updated very infrequently, so the scraper runs every 6 months to replenish the existing csv file in an S3 bucket.

![image](https://github.com/user-attachments/assets/d3dee3ff-cdbc-4048-8f80-8ae997cec4df)

In most cases, the Lambda function that handles this process searches the CSV file for the given course's C-ID. 

```js
const records = parse(csvData, {
  columns: true, 
  skip_empty_lines: true,
});

function findCid(records, deptName, deptNum, courseTitle) {
  const searchColumn = "Course Title"; 
  
  const entry = records.filter(record => record[searchColumn] === courseTitle)[0]; // gets newest entry
  
  let cid;
  
  if (entry && entry["Dept Name"] === deptName && entry["Dept Number"] === deptNum) {
    cid = entry["C-ID #"];
  }
  
  return cid;
}
```

The above logic is then applied to an entire list of articulations and replaces the same articulations without C-IDs in DynamoDB.

```js
export function appendCids(courseData, records) {
  for (let i = 0; i < courseData.length; ) {
    const item = courseData[i];
      
    if (item.result) {
      appendCids(item.result, records);
    }
        
    if (Array.isArray(item)) {
      appendCids(item, records);
    } 
      
    if (item && item.prefix && item.courseNumber && item.courseTitle) {
      const { prefix, courseNumber, courseTitle } = item;
        
      const cid = findCid(records, prefix, courseNumber, courseTitle);

      if (cid) {
        item.cid = cid;
      } else {
        console.warn(`no match found for ${prefix} ${courseNumber} - ${courseTitle}`);
      }
    } 
      
    i += 1;
  }
    
  return courseData;
}
```

The new articulation data with C-ID properties can now be viewed on the frontend with a toggle switch.

## Problems with C-IDs

From extensive research, it can be determined that ASSIST.org updates their data more than C-ID.net. This increases the chance that a certain course may not have a corresponding C-ID in the master csv file.

Course titles, department numbers, and department prefixes may also be different than what is in the ASSIST.org database. An extra space in any of these fields could jeopardize the logic of ```findCid()```.

----
