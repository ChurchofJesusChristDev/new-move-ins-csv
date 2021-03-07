# New Move-ins to CSV

A JavaScript Console Script to get New Move Ins as a CSV / TSV

> As an Elder's Quorum we want to review new move-ins on a regular basis.
> Since our ward is very transient, we sometimes have many new ward members.
> 
> This script helps us by scraping existing table and adding the neccessary contact information -
> which is not available in the default view -
> so that we can quickly and easily send out welcome messages.

## Usage

Works when logged into LCR and navigated to the Recent Move-Ins Page: \
<https://lcr.churchofjesuschrist.org/report/members-moved-in?lang=eng>

Copy and run script below in Safari's (or any browser's) JavaScript console to get new members in tab-separated format:

```txt
[23 Dec 2020]	(___) ___-____	32	F	"Jane Mary Doe" <null@example.com>,
```

This can be pasted into a Spreadsheet, but is also formatted well enough for email.

```js
function $(sel, el) {
  if (!el) {
    el = document.body;
  }
  return el.querySelector(sel);
}

function getNewMoveIns() {
  let rows = $$("tr");
  return Promise.all(
    rows.slice(1).map(async function (el) {
      //console.log(el.innerText);
      var member = memberToJSON(el);

      var memberId = member.memberUrl.replace(
        /.*member-profile\/([^\?]+).*/,
        "$1"
      );
      var emailUrl =
        "https://lcr.churchofjesuschrist.org/services/member-card?id=" +
        memberId +
        "&includePriesthood=true&lang=eng&type=INDIVIDUAL";
      //console.log("id:", memberId);
      var mem = await fetch(emailUrl, { credentials: "same-origin" }).then(
        function (resp) {
          return resp.json().then(function (data) {
            return data;
          });
        }
      );
      member.email = mem.email;
      member.birthDate = mem.birthDate;
      return member;
    })
  );
}

function formatTSV(m) {
  return (
    "[" +
    m.moveDate.padStart(11, "0") +
    "]" +
    "\t" +
    formatTel(m.tel) +
    "\t" +
    m.age +
    "\t" +
    m.sex +
    "\t" +
    '"' +
    m.first.split(/, /g).reverse().join(" ") +
    '" <' +
    m.email +
    ">," +
    "\t"
  );
}

function memberToJSON(el) {
  return {
    memberUrl: $("a", el).href,
    first: $("td.first", el).innerText,
    moveDate: $("td.move-date", el).innerText,
    age: $("td.age", el).innerText,
    sex: $("td.sex", el).innerText,
    adr: $("td.adr", el).innerText,
    tel: $("td.tel", el).innerText,
  };
}

function formatTel(t) {
  t = t.replace(/\D/g, "");
  if (t.length > 10) {
    console.warn("weird tel?", t);
    t = t.slice(-10);
  }
  if (t.length < 10) {
    t = t.padStart(10, "_");
  }
  return `(${t.slice(0, 3)}) ${t.slice(3, 6)}-${t.slice(6, 10)}`;
}

getNewMoveIns().then(function (tsv) {
  let text = tsv
    .map(function (m) {
      //if ('M' != m.sex) {
      //    console.log("Skipping female", JSON.stringify(m));
      //    return;
      //}
      if (!(m.age > 17)) {
        console.warn("Skipping minor", JSON.stringify(m));
        return "";
      }
      return formatTSV(m);
    })
    .filter(Boolean)
    .join("\n");

  console.info(text);
});
```
