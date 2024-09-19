---
outline: deep
---

# Create Dynamic Calindar Flex Message 
ระบบนี้ เป็นตัวอย่างพัฒนาโดยใช้ `Google app script` โดยเรามีการบันทึกกิจกรรมไว้ล่วงหน้าภายใน `Google Sheet`
เริ่มต้นเขียนฟังก์ชั่นเพื่อรอรับค่าจากข้อความที่ถูกส่งเข้ามา ผ่าน Line messaging api เช่นในส่วนของ handleMessage ฟังก์ชั่นให้เรียกใช้ฟังก์ชั่น startCreateCalendar() โดยส่งอากิวเมนต์ ได้แก่ ตัวเลขของ ปี และ ตัวเลขของเดือน ที่ต้องการตามลำดับจากนั้นจะทำการ สร้างเป็น Object JSON Flex เพื่อตอบกลับเป็นรูปแบบปฏิทินโดยแสดงกิจกรรมในวันต่างๆที่ระบุใน Sheet 

## Example Column on Sheet "กิจกรรม"
| วันที่        | กิจกรรม          |
|--------------|-------------------|
| 15/8/2024    | ทดสอบกิจกรรม      |
| 17/7/2024    | ทดสอบกิจกรรม      |
| 17/7/2024    | ทดสอบกิจกรรม3     |


## Function for CreateCalendar in your function HandleMessage()
จากตัวอย่างนี้จะรับข้อความแล้วนำมาเปรียบเทียบโดยใช้ regular expression ร่วมในการเช็คข้อความที่ผู้ใช้ส่งเข้ามาที่ได้รับจาก Event message ว่ามีคำขึ้นต้นว่า calendar หรือไม่ เช่นข้อความที่ได้รับ เป็น calendar/2024/8 ก็จะหมายถึง ส่งค่าปีเป็น คศ.2024 และเดือนเป็นเดือนสิงหาคม เพื่อระบุสร้าง Flex ของเดือน สิงหาคม ค.ศ.2024 
````JS
if (userMSG.match(/^calendar/) !== null) {
  let [, txtSearch1, txtSearch2] = userMSG.split("/");
  msg = testcalendar(parseInt(txtSearch1), parseInt(txtSearch2));
}
````
## Function main
````JS
function startCreateCalendar(year, month) {
  try {
    let flex = createCalendarFlexMessage(year, month);
    Logger.log(JSON.stringify(flex));
    return [flex];
  } catch (error) {
    Logger.error("Error: " + error.message);
    return ["Error: " + error.message];
  }
}
````
## set Function For Create Flex
เราจะสร้างฟังก์ชัน `createCalendarFlexMessage` เพื่อสร้างปฏิทินแบบ Flex Message สำหรับการส่งผ่าน LINE API โดยดึงข้อมูลกิจกรรมจาก Google Sheet และแสดงในปฏิทิน

## 1. ตรวจสอบเดือน
ฟังก์ชันนี้เริ่มด้วยการรับค่า year และ month ถ้าไม่ระบุค่าจะใช้ปีและเดือนปัจจุบัน จากนั้นตรวจสอบว่าเดือนที่รับมานั้นอยู่ในช่วง 1 ถึง 12 หรือไม่ หากไม่ถูกต้องจะโยนข้อผิดพลาด
````JS
function createCalendarFlexMessage(year = new Date().getFullYear(), month = new Date().getMonth() + 1) {
  if (month < 1 || month > 12) {
    throw new Error("Invalid month. Must be between 1 and 12.");
  }
````



## 2. คำนวณจำนวนวันและตำแหน่งวันแรก
จำนวนวันในเดือน (daysInMonth)
วันที่เริ่มต้นของเดือน (firstDayOfMonth)
จำนวนวันของเดือนก่อนหน้า (lastDayOfPreviousMonth)

````JS
  const daysInMonth = new Date(year, month, 0).getDate();
  const firstDayOfMonth = new Date(year, month - 1, 1).getDay();
  const lastDayOfPreviousMonth = new Date(year, month - 1, 0).getDate();

````

## 3. ดึงข้อมูลกิจกรรมจาก Google Sheet
แปลงชื่อเดือนเป็นข้อความ (monthNames)
ดึงข้อมูลจาก Google Sheet โดยคอลัมน์แรกเป็นวันที่ และคอลัมน์ที่สองเป็นกิจกรรม
กรองข้อมูลที่ตรงกับเดือนและปีที่ระบุ
````JS
  const monthNames = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];
  const monthName = monthNames[month - 1];

  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('กิจกรรม');
  const range = sheet.getRange('A:B').getDisplayValues();
  const events = range.reduce((acc, [dateStr, event]) => {
    if (dateStr) {
      const [day, m, y] = dateStr.split('/').map(Number);
      if (y === year && m === month) {
        if (!acc[day]) acc[day] = [];
        acc[day].push(event);
      }
    }
    return acc;
  }, {});

````

## 4. สร้างแถว (weekRow) ที่เก็บวันทั้ง 7 ของสัปดาห์ และเพิ่มเข้าไปใน calendarRows ซึ่งใช้เก็บข้อมูลทั้งหมดของปฏิทิน

````JS
  let calendarRows = [];
  let weekRow = [];

  const weekdays = ['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'];
  weekRow = weekdays.map(day => ({ "type": "text", "text": day, "weight": "bold", "size": "sm" }));
  calendarRows.push(weekRow);


````

## 5. แสดงวันของเดือนก่อนหน้า
ถ้าเดือนนี้ไม่เริ่มที่วันอาทิตย์ จะเติมวันของเดือนก่อนหน้าในแถวแรกให้ครบ 7 วัน
````JS
  weekRow = new Array(7).fill({ "type": "text", "text": "", "align": "center" });
  let day = 1;

  for (let i = firstDayOfMonth - 1; i >= 0; i--) {
    weekRow[i] = { "type": "text", "text": String(lastDayOfPreviousMonth - (firstDayOfMonth - 1 - i)), "align": "center", "color": "#AAAAAA" };
  }


````
## 6. วนลูปสร้างวันของเดือน
แสดงวันที่ของเดือน โดยเติมวันในสัปดาห์แต่ละแถวให้ครบ 7 วัน หากมีเหตุการณ์ในวันนั้นจะเปลี่ยนสีของวันเป็นสีชมพู

````JS
  for (; day <= daysInMonth; day++) {
    let cellStyle = { "type": "text", "text": String(day), "align": "center" };
    if (events[day]) {
      cellStyle = {
        "type": "text",
        "text": String(day),
        "align": "center",
        "color": "#FF69B4"
      };
    }
    weekRow[(firstDayOfMonth + day - 1) % 7] = cellStyle;
    if ((firstDayOfMonth + day) % 7 === 0 || day === daysInMonth) {
      calendarRows.push(weekRow);
      weekRow = new Array(7).fill({ "type": "text", "text": "", "align": "center" });
    }
  }


````

## 7. เติมวันของเดือนถัดไป
หากแถวสุดท้ายไม่เต็ม 7 วัน จะเติมวันของเดือนถัดไปให้เต็มแถว
````JS
  let lastRow = calendarRows[calendarRows.length - 1];
  let lastRowFilled = lastRow.filter(cell => cell.text !== "").length;

  if (lastRowFilled < 7) {
    for (let i = 1; lastRowFilled < 7; i++, lastRowFilled++) {
      lastRow[lastRowFilled] = {
        "type": "text",
        "text": String(i),
        "align": "center",
        "color": "#AAAAAA"
      };
    }
  }
````

## 8. สร้าง Flex Message สำหรับปฏิทิน
สุดท้ายสร้าง Flex Message โดยใช้ข้อมูลที่เตรียมไว้ เช่น วันในปฏิทินและกิจกรรม โดยมีส่วน header เป็นชื่อเดือนและปี และส่วน body เป็นตารางปฏิทินพร้อมกับกิจกรรมในวันต่าง ๆ

````JS
  const flexMessage = {
    "type": "flex",
    "altText": "Calendar for " + month + "/" + year,
    "contents": {
      "type": "bubble",
      "size": "giga",
      "header": {
        "type": "box",
        "layout": "vertical",
        "contents": [
          {
            "type": "text",
            "text": `${monthName} ${year}`,
            "weight": "bold",
            "size": "xl",
            "align": "center"
          }
        ]
      },
      "body": {
        "type": "box",
        "layout": "vertical",
        "contents": [
          ...calendarRows.map(row => ({
            "type": "box",
            "layout": "baseline",
            "contents": row,
          })),
          {
            "type": "box",
            "layout": "vertical",
            "contents": eventDetails,
            "margin": "lg",
            "paddingStart": "15px",
          }
        ]
      }
    }
  };

  return flexMessage;
}
````

