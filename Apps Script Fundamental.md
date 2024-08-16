<h1 style="display: flex; justify-content: space-between; align-items: center;">
    Apps Script with Google Workspace APIs Fundamental - Examples
    <img src="CloudMile-logo.gif" alt="GIF" style="height: 2em;" />
</h1>

- [Gmail](#gmail)
  - [Query mail threads from Gmail](#query-mail-threads-from-gmail)
  - [Send an email with string body](#send-an-email-with-string-body)
  - [Send an email with HTML body](#send-an-email-with-html-body)
- [Google Calendar](#google-calendar)
  - [Get Calendar events](#get-calendar-events)
  - [Create a Calendar event](#create-a-calendar-event)
  - [Get all Calendars that the user can access](#get-all-calendars-that-the-user-can-access)
- [Google Drive](#google-drive)
  - [Get all files](#get-all-files)
  - [Remove untitled files](#remove-untitled-files)
  - [Create a Folder, grant access to the folder and move a file to the folder](#create-a-folder-grant-access-to-the-folder-and-move-a-file-to-the-folder)
- [Google Sheets](#google-sheets)
  - [Get all sheets](#get-all-sheets)
  - [Get sheet data](#get-sheet-data)
  - [Write data to sheet](#write-data-to-sheet)
- [Properties Service](#properties-service)
  - [Get Script Properties](#get-script-properties)
  - [Set Script Properties](#set-script-properties)
---

## Gmail
<em>Reference: [https://developers.google.com/apps-script/reference/gmail](https://developers.google.com/apps-script/reference/gmail)</em>

### Query mail threads from Gmail
```javascript
function queryGmail() {
  const senderEmailAddress = "tri-thong.tran@mile.cloud";
  const subjectToSearch = "Hello GWS!";
  const minNum = 0;
  const maxNum = 5;
  var threads = GmailApp.search(
    `from:"${senderEmailAddress}",subject:"${subjectToSearch}"`,
    minNum,
    maxNum
  );
  for (let i = 0; i < threads.length; i++) {
    const threadId = threads[i].getId()
    const firstMessageSubject = threads[i].getFirstMessageSubject()
    Logger.log("Thread ID: " + threadId);
    Logger.log("First Message Subject: " + firstMessageSubject);
    Logger.log("------------------------------------------------");
  }
}
```

### Send an email with string body
```javascript
function sendGmail() {
  const now = new Date();
  const senderName = "Tri-Thong";
  const receiverEmailAddress = "tri-thong.tran@mile.cloud";
  const ccPeople = "trithong12@gmail.com,trithong12vn@gmail.com";
  const mailSubject = "Hello GWS! - Current Time";
  const mailStrBody = "The time is: " + now.toString();
  GmailApp.sendEmail(
    receiverEmailAddress,
    mailSubject,
    mailStrBody,
    {
      name: senderName,
      cc: ccPeople
    }
  );
}
```

### Send an email with HTML body
```javascript
function sendGmail() {
  const now = new Date();
  const senderName = "Tri-Thong";
  const receiverEmailAddress = "tri-thong.tran@mile.cloud";
  const ccPeople = "trithong12@gmail.com,trithong12vn@gmail.com";
  const mailSubject = "Hello GWS! - Current Time";
  const mailStrBody = "The time is: " + now.toString();
  const mailHtmlBody = `
  <!DOCTYPE html>
  <html lang="en">
    <title>Page Title</title>
    <body>
      <div>
        <h1>This is a Heading</h1>
        <p>This is a paragraph.</p>
        <p>This is another paragraph.</p>
      </div>
    </body>
  </html>
  `;
  GmailApp.sendEmail(
    receiverEmailAddress,
    mailSubject,
    mailStrBody,
    {
      name: senderName,
      cc: ccPeople,
      htmlBody: mailHtmlBody
    }
  );
}
```

---

## Google Calendar
<em>Reference: [https://developers.google.com/apps-script/reference/calendar](https://developers.google.com/apps-script/reference/calendar)</em>

### Get Calendar events
```javascript
function getCalendarEvents() {
  const now = new Date();
  const twoHoursFromNow = new Date(now.getTime() + (2 * 60 * 60 * 1000)); // constructor Date(milliseconds)
  const events = CalendarApp.getDefaultCalendar().getEvents(now, twoHoursFromNow);
  for (var i = 0; i < events.length; i++) {
    const eventTitle = events[i].getTitle();
    Logger.log("Event title: " + eventTitle);
  }
}
```

### Create a Calendar event
```javascript
function createCalendarEvent() {
  var event = CalendarApp.getDefaultCalendar().createEvent(
    'Apps Script Workshop',
    new Date('2024-08-16T15:00:00'),
    new Date('2024-08-16T17:00:00'),
    {
      guests: "tri-thong.tran@mile.cloud",
      description: "This is a workshop for college students, aims to introduce Google Apps Script.",
      sendInvites: false,
      location: "TW-Office-33-雅加達CGK"
    }
  );
  Logger.log('Event ID: ' + event.getId());
}
```

### Get all Calendars that the user can access
```javascript
function getAllCalendars() {
  // Determines how many calendars the user can access.
  var calendars = CalendarApp.getAllCalendars();
  for (var i = 0; i < calendars.length; i++) {
    const calendarName = calendars[i].getName();
    Logger.log(calendarName);
  }
}
```

---

## Google Drive
<em>Reference: [https://developers.google.com/apps-script/reference/drive](https://developers.google.com/apps-script/reference/drive)</em>
### Get all files
```javascript
function getFiles() {
  // Logs the name of every file in the user's Drive.
  var files = DriveApp.getFiles();
  var i = 0;
  while (files.hasNext() && i < 10) {
    var file = files.next();
    console.log(file.getName());
    i++;
  }
}

function getFullPath(file) {
  var name = file.getName();
  var parents = file.getParents();
  
  // Build the full path
  var path = name;
  while (parents.hasNext()) {
    var parent = parents.next();
    path = parent.getName() + '/' + path;
    parents = parent.getParents();
  }
  
  return path;
}
```

### Remove untitled files
```javascript
function removeUntilteFiles() {
  // Trash every untitled spreadsheet that hasn't been updated in a week.
  const currentUser = Session.getActiveUser().getEmail();
  var files = DriveApp.getFilesByName('Untitled document');
  var count = 0;
  while (files.hasNext()) {
    var file = files.next();
    const isOwner = (file.getOwner() && currentUser == file.getOwner().getEmail());
    
    if (isOwner) {
      console.log("1 file found");
      // file.setTrashed(true);
      count++;
    }
  }
  console.log("Totally removed " + count + " file(s).");
}
```

### Create a Folder, grant access to the folder and move a file to the folder
```javascript
function main() {
  folder = createFolder("Demo folder");
  file = createFile("Text File.txt", "Hello, Google Apps Script!");
  addFileIntoFolder(folder, file);
  grantFolderPermission(folder, ["trithong12@gmail.com"], "Viewer");
}

function createFile(fileName, content) {
  return DriveApp.getRootFolder().createFile(fileName, content);
}

function createFolder(folderName) {
  const folders = DriveApp.getFoldersByName(folderName);
  if (!folders.hasNext()) {
    return DriveApp.createFolder(folderName);
  }
  return DriveApp.createFolder(folderName + "_" + new Date().toUTCString());
}

function grantFolderPermission(folder, people, permission) {
  switch(permission) {
    case "Viewer": folder.addViewers(people); break;
    case "Editor": folder.addEditors(people); break;
  }
}

function addFileIntoFolder(folder, file) {
  file.moveTo(folder);
}
```

---

## Google Sheets
<em>Reference: [https://developers.google.com/apps-script/reference/spreadsheet](https://developers.google.com/apps-script/reference/spreadsheet)</em>
### Get all sheets
```javascript
function listAllSheets() {
  const spreadSheetUrl = "https://docs.google.com/spreadsheets/d/1Md9HX6fHkG_OGVpveXPpKm63KeGEuZF9JbdTADu5DSw/edit?usp=sharing";
  const spreadSheet = SpreadsheetApp.openByUrl(spreadSheetUrl);
  const sheets = spreadSheet.getSheets();
  for (let i = 0; i < sheets.length; i++) {
    console.log(sheets[i].getName());
  }
}
```

### Get sheet data
```javascript
function getGoogleSheetData() {
  const spreadsheetUrl = "https://docs.google.com/spreadsheets/d/1Md9HX6fHkG_OGVpveXPpKm63KeGEuZF9JbdTADu5DSw/edit?gid=0#gid=0";
  const data = SpreadsheetApp
    .openByUrl(spreadsheetUrl)
    .getActiveSheet()
    .getDataRange()
    .getValues();
  console.log(data);
  return data;
}
```

### Write data to sheet
```javascript
function writeCells() {
  const spreadsheetUrl = "https://docs.google.com/spreadsheets/d/1Md9HX6fHkG_OGVpveXPpKm63KeGEuZF9JbdTADu5DSw/edit?gid=0#gid=0";
  var sheet = SpreadsheetApp.openByUrl(spreadsheetUrl).getActiveSheet();
  var date = new Date();

  // getRange(a1Notation) - a single cell
  sheet.getRange("A1").setValue(date);

  // getRange(a1Notation) - a range
  sheet.getRange("A3:B4").setValues([
    ["東", "南"],
    ["西", "北"]
  ]);

  // getRange(rowNumber, colNumber, numOfRows, numOfCols)
  sheet.getRange(6, 1, 3, 3).setValues([
    ["A", "B", "C"],
    ["D", "E", "F"],
    ["G", "H", "I"]
  ])
}
```
---
## Properties Service
### Get Script Properties
```javascript
function getScriptProperty() {
  // Get the ScriptProperties object
  var scriptProperties = PropertiesService.getScriptProperties();
  
  // Retrieve a script-wide property
  var value = scriptProperties.getProperty('myKey');
  
  Logger.log('Retrieved script property: myKey = ' + value);
}
```

### Set Script Properties
```javascript
function setScriptProperty() {
  // Get the ScriptProperties object
  var scriptProperties = PropertiesService.getScriptProperties();
  
  // Set a script-wide property
  scriptProperties.setProperty('myKey', 'myValue');
  
  Logger.log('Script property set: myKey = myValue');
}
```