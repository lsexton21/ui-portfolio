## Handling And Validating Large PDF Imports

This is a tool I was proud to make and one where I had my first encounter with AWS services.  I'm proud of the UI because I added some more complex validations for the PDF document itself, which required for the browser to stream-read a chunk of the PDF.  The backend contained some lamdas that were organized by an AWS step-function (which, being my first encounter with AWS, I co-developed with a senior backend dev)

**See below for code**

![Handling PDFs](/assets/bulkJobs.png)

---


#### Below is a method executed when uploading a PDF file.  In it, I put together a validation to make sure that the PDF file contents did not start with UTF-8 BOM characters which is created by some processors.  These characters prevented our backend functions from processing the PDF correctly.  It would be crazy to force the front end to parse the whole PDF which JavaScript PDF parsers do; thus, I created my own file reader stream, and only parsed the first chunk because that is all I needed.

```
_importCSV(type) {
    const org = App.getOrgFilter() ? App.getOrgFilter() : App.getOrganization();

    MessageModal.show({
      title: type === "add" ? "Add from CSV" : "Update from CSV",
      body:
        `<p>Please note that you are currently importing numbers as <strong>${org.getName()}</strong>.` +
        (App.accountIsAdmin()
          ? "<br /><br /><em>*If you wish to import numbers as a different organization, please change your organization filter at the top of the screen.</em></p>"
          : ""),
      confirmBtn: "Import From CSV",
      addCallback: () => {
        const fileSelector = new FileSelector();
        fileSelector.selectFile(".csv", async (file) => {
          const fileBlob = new Blob([file], { type: "text/csv" });
          const fileBlobStream = fileBlob.stream();
          const fileReader = fileBlobStream.getReader();

          const logStartDate = new Date();
          const { value } = await fileReader.read();
          const chunk = new TextDecoder().decode(value);

          if (isDevelopment) {
            const logEndDate = new Date();
            console.log(
              "Total Processing Time: " +
                (logEndDate.getTime() - logStartDate.getTime()) +
                "ms"
            );
            console.log(
              "Total Characters Processed: " + chunk.length + " characters"
            );
          }

          if (!this.validateHeaders(type, chunk)) {
            return false;
          }

          if (chunk.charCodeAt(0) === 0xFEFF) {
            MessageModal.show({
              title: "Import Error",
              text: "The CSV file you are trying to import was created by an unrecognized processor. Please use another processor to create your CSV file.",
              isError: true,
              devErrorMsg:
                "A UTF-8 BOM was detected by the file reader.  The server can not handle this type of BOM.  Instruct the user to create their CSV file with a different processor.",
            });
          } else {
            if (type === "add") {
              this.sendRequests(type, file);
            } else {
              this._bulkUpdateEditorModal.updateFileName(file.name);
              this._bulkUpdateEditorModal.showAddModal(null, (importData) => {
                this.sendRequests(type, file, importData);
              });
            }
          }
        });
      },
    });
  }
```

---

#### Since I was parsing the first chunk anyway (see above), I also added logic to validate the headers of the PDF document here:

```
validateHeaders(type, chunk) {
    const csvHeaders = chunk
      .split("\n")[0]
      .replace(/(\r\n|\n|\r)/gm, "")
      .split(",");

    let missingHeaders = [];
    let duplicatedHeaders = [];
    let emptyColumns = 0;
    let invalidHeaders = [];

    if (type === "add") {
      BulkJobsPage.requiredAddHeaders.forEach((header) => {
        if (!csvHeaders.includes(header)) {
          missingHeaders.push(header);
        }
      });
      csvHeaders.forEach((header) => {
        if (!BulkJobsPage.validAddHeaders.includes(header)) {
          invalidHeaders.push(header);
        }
      });
    } else {
      BulkJobsPage.requiredUpdateHeaders.forEach((header) => {
        if (!csvHeaders.includes(header)) {
          missingHeaders.push(header);
        }
      });
      csvHeaders.forEach((header) => {
        if (!BulkJobsPage.validUpdateHeaders.includes(header)) {
          invalidHeaders.push(header);
        }
      });
    }

    for (let i = 0; i < csvHeaders.length; i++) {
      for (let j = i + 1; j < csvHeaders.length; j++) {
        if (csvHeaders[i] === csvHeaders[j]) {
          duplicatedHeaders.push(csvHeaders[i]);
        }
      }
    }

    for (let header of csvHeaders) {
      if (header === "") {
        emptyColumns++;
      }
    }

    if (
      missingHeaders.length > 0 ||
      duplicatedHeaders.length > 0 ||
      emptyColumns > 0 ||
      invalidHeaders.length > 0
    ) {
      MessageModal.show({
        title: "Import Error",
        body:
          "<div><p>The CSV file you are trying to import:<p><br>" +
          (missingHeaders.length > 0
            ? `<p>Is missing the following headers: <strong>${missingHeaders.join(
                ", "
              )}</strong></p>`
            : "") +
          (duplicatedHeaders.length > 0
            ? `<p>Contains the following duplicated headers: <strong>${duplicatedHeaders.join(
                ", "
              )}</strong></p>`
            : "") +
          (emptyColumns > 0
            ? `<p>Has <strong>${emptyColumns}</strong> empty column${
                emptyColumns > 1 ? "s" : ""
              }</p>`
            : "") +
          (invalidHeaders.length > 0
            ? `<p>Contains the following invalid headers: <strong>${invalidHeaders.join(
                ", "
              )}</strong></p>`
            : "") +
          "<br>" +
          "<p><em>If you feel that this is a mistake, please contact Skyetel support.</em></p></div>",
        isError: true,
        devErrorMsg: "Header Validation Error",
      });
      return false;
    }

    return true;
  }
  ```

  [Return to main menu](../README.md)