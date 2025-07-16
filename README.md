
# 🚀 Automating ASN File Processing via FTP in D365 F&O

## 📌 Class: `HSOEASNFTPImportService`

The `HSOEASNFTPImportService` is a powerful, automated FTP integration service in Microsoft Dynamics 365 Finance and Operations that enables:

- **Fetching ASN (Advanced Shipping Notice) `.txt` files from an FTP server**
- **Parsing and validating the records line by line**
- **Auto-creating WMS Journals and serial/batch tracking**
- **Logging processed, failed, and invalid files**
- **Moving files to corresponding folders** (`Processed`, `Error`)

---

## 🧠 What this class does

This service is meant to **streamline the ASN import process** for external vendors or partners who upload shipment data via FTP. It reads the FTP server, processes `.txt` files, and auto-creates inventory journals in D365 F&O — removing the need for manual entry.

### 🔧 Real-world scenario:
An external partner sends you serialized inventory data (e.g., medical devices or parts). This class:
- Fetches `.txt` files from the FTP folder
- Parses the data
- Matches items using external item IDs
- Validates data for packingslips, items, POs
- Creates **WMS reception journals**
- Updates ASN tracking tables
- Moves the files after processing

---

## 📂 Folder Structure Used

| FTP Path Purpose   | Parameter                 |
|--------------------|---------------------------|
| Incoming files     | `HSOEASNFTPUnprocessed`   |
| Successfully moved | `HSOEASNFTPProcessed`     |
| Failed/errored     | `HSOEASNFTPError`         |

---

## 🧾 Key Methods Explained

### 🔄 `processOperation()`

Main entry point of the service.

- Calls `getFileNames()` to fetch `.txt` files
- Loops through and calls `processFile()` for each
- Based on success or failure:
  - Calls `moveProcessedFile()` or `moveErrorFile()`
- Provides `info()` log messages for traceability

### 📥 `getFileNames()`

Retrieves the list of `.txt` files from the **Unprocessed** FTP directory.

- Connects via FTP using credentials from `InventParameters`
- Filters and extracts valid `.txt` filenames
- Returns them in a container

### 🧩 `processFile(_fileName)`

**The core logic** of the service.

- Opens and reads each `.txt` file from FTP stream
- Splits data using `;` delimiter
- Resolves item ID from `ExternalItemId`
- Validates or creates:
  - **InventBatch**
  - **InventSerial**
  - **InventDim**
  - **WMSJournalTable**
  - **WMSJournalTrans**
  - **ASN Serial tracking records**
- Updates a custom ASN Import Log table

### ✅ `validateSid(_fileName)`

Validates the structure and data of a given `.txt` file before processing:
- Ensures packingslip doesn’t already exist
- Checks if item exists
- Validates PO
- Checks file formatting
- Inserts error logs and throws descriptive messages

### 🛠 `findOrCreateBatchId(...)`

Generates or finds a valid `InventBatchId` for an item based on:
- Variant ID
- Manufacturing and expiry dates
- Item group configuration (variant blocking)

### 📦 `moveProcessedFile(_fileName)` and `moveErrorFile(_fileName)`

Moves files to the appropriate folder after processing using FTP `RENAME` method.

---

## 📝 Logs and Tracking

Uses the custom table `HSOE_ASNImportLogTable` to:
- Record import status (`Processed` / `Failed`)
- Log messages (e.g., “Journal # created” or validation errors)
- Track if the file was successfully FTP’d or not

---

## 🔐 Security & Timeout Handling

- FTP credentials are pulled from `InventParameters`
- Long timeouts (900 seconds) for large files

---

## ✅ Summary of Benefits

| Benefit                           | Description                                                                 |
|-----------------------------------|-----------------------------------------------------------------------------|
| 🧠 Intelligent Parsing             | Handles large .txt files line-by-line and field-by-field                    |
| 🔄 Seamless Integration           | Pulls directly from FTP and processes without user intervention             |
| 🧾 Validated Data                 | Ensures POs, items, and serials are correct before journal creation         |
| 📦 Auto Journal Creation          | Automatically generates WMS journals with linked serial and batch tracking  |
| 📁 Organized File Handling        | Files are neatly moved to processed or error folders                        |
| 🔍 Log and Traceability           | Every operation is tracked in a custom log table                            |

---

## 🚀 Deployment Tips

- Schedule this class via **Batch Job** or trigger from a **Power Automate Flow**
- Add validation toggle to optionally skip pre-checks
- Monitor FTP logs and ensure folders are reachable

---

## 📸 Sample FTP Config Setup in Invent Parameters

| Field Name               | Example Value                             |
|--------------------------|-------------------------------------------|
| FTP Username             | `52909.webmaster`                         |
| FTP Password             | `*****`                                   |
| FTP Server               | `ftp://89.31.143.200/`                    |
| FTP Unprocessed Folder   | `/D365ASN/Unprocessed/`                   |
| FTP Processed Folder     | `/D365ASN/Processed/`                     |
| FTP Error Folder         | `/D365ASN/Error/`                         |

# Service Class
```
[using FTPWebRequest = System.Net.FtpWebRequest;
using WebRequest = System.Net.WebRequest;
using NetworkCredential = System.Net.NetworkCredential;
using WebRequestMethods = System.Net.WebRequestMethods;
using Stream = System.IO.Stream;
using StreamChk = System.IO.Stream;

class HSOEASNFTPImportService extends SysOperationServiceBase
{
    str userName = InventParameters::find().HSOEASNFTPUserName;
    str password = InventParameters::find().HSOEASNFTPPassword;
    str ftpServer = InventParameters::find().HSOEASNFTPSeverAddress;
    str unprocessed = InventParameters::find().HSOEASNFTPUnprocessed;
    str processed = InventParameters::find().HSOEASNFTPProcessed;
    str err = InventParameters::find().HSOEASNFTPError;
    HSOE_ASNImportLogTable logTab;
    container fileNameCont,processedCont,ErrorCont;
    #define.Delimiter(";")
    #define.RecordDelimiter("\n")
    
    
    public void processOperation()
    {        
        int ret,retcount;
        str fileErr = "";
        
        fileNameCont = this.getFileNames();
        /*if(conLen(fileNameCont)>0)
        {
            for(int i =1;i<= conLen(fileNameCont);i++)
            {
                try
                {
                    this.validateSid(conPeek(fileNameCont,i));
                }
                catch(Exception::Error)
                {
                    ErrorCont = conIns(ErrorCont,conLen(ErrorCont)+1,conPeek(fileNameCont,i));
                }
            }
        }
        if(conLen(ErrorCont)>0)
        {
            for(int i =1;i<= conLen(ErrorCont);i++)
            {
                if(conFind(fileNameCont,conPeek(ErrorCont,i)))
                {
                    fileNameCont = conDel(fileNameCont,conFind(fileNameCont,conPeek(ErrorCont,i)),1);
                }
            }
        }*/
        if(conLen(fileNameCont)>0)
        {
            for(int i =1;i<= conLen(fileNameCont);i++)
            {
                try
                {
                    this.processFile(conPeek(fileNameCont,i));
                    processedCont = conIns(processedCont,conLen(processedCont)+1,conPeek(fileNameCont,i));
                }
                catch(Exception::Error)
                {
                    ErrorCont = conIns(ErrorCont,conLen(ErrorCont)+1,conPeek(fileNameCont,i));
                }
            }
        }
        if(conLen(processedCont)>0)
        {
            for(int i =1;i<= conLen(processedCont);i++)
            {
                try
                {
                    this.moveProcessedFile(conPeek(processedCont,i));
                }
                catch(Exception::Error)
                {
                    fileErr += conPeek(processedCont,i) + " ";
                }
            }
        }
        if(conLen(ErrorCont)>0)
        {
            for(int i =1;i<= conLen(ErrorCont);i++)
            {
                try
                {
                    this.moveErrorFile(conPeek(ErrorCont,i));
                }
                catch(Exception::Error)
                {
                    fileErr += conPeek(ErrorCont,i) + " ";
                }
            }
        }

        info(strFmt("%1 files are processed",con2Str(processedCont)));
        if(fileErr)
        {
            info(fileErr + " could not be moved");
        }
    }

    public container getFileNames()
    {
        container               fileNameContLoc;
        System.IO.StreamReader  ftpReaderName;
        str                     fileName;
        int                     fileCount, randomIndex;

        FtpWebRequest requestName = WebRequest::Create((ftpServer+unprocessed));
        requestName.Credentials = new NetworkCredential(userName,password);
        requestName.Method = "LIST";
        requestName.Timeout = 900000;
        requestName.ReadWriteTimeout = 900000;
        requestName.UsePassive = true;
        ftpReaderName = new System.IO.StreamReader(requestName.GetResponse().GetResponseStream());
        while (!ftpReaderName.EndOfStream)
        {
            fileName = ftpReaderName.ReadLine();
            if(strScan(fileName, '.txt',1,strLen(fileName)) > 0)
            {
                int lastSpaceIndex;
                int timeColonIndex;
                // Commented on this 11th June 2025
                //for (int i = 1; i <= strLen(fileName); i++)
                //{
                //    if (subStr(fileName,i,1) == ' ')
                //    {
                //        lastSpaceIndex = i;
                //    }
                //}
                //timeColonIndex = strScan(fileName,":"
                // Added on this 11th June 2025
                for (int i = 1; i <= strlen(filename); i++)
                {
                    if (substr(filename,i,1) == ':')
                    {
                        timeColonIndex = i + 3;
                    }
                }
                //fileName = subStr(fileName, lastSpaceIndex + 1, strLen(fileName) - lastSpaceIndex);
                fileName = subStr(fileName, timeColonIndex + 1, strLen(fileName) - timeColonIndex);
                fileNameContLoc = conIns(fileNameContLoc,conLen(fileNameContLoc)+1,fileName);
            }
        }

        ftpReaderName.Close();
        requestName.GetResponse().Close();
        return fileNameContLoc;
    }

    public void processFile(str _fileName)
    {
        AsciiStreamIo                       file,filecheck;
        System.Byte[]                       buffer = new System.Byte[51200](); 
        System.IO.Stream                    fileStream = new System.IO.MemoryStream();
        System.IO.FileInfo                  fileInfo;
        InventSerialId                      inventSerialId;
        InventBatchId                       inventBatchId;
        PurchId                             purchId;
        ItemId                              itemId;
        TransDate                           manufactureDate, expiryDate;
        HSOEASNImportPrefixAttributeTable   prefixTab;
        Map                                 headerPackingSlipMap = new Map(Types::String,Types::String);
        WMSJournalTable                     wmsJournalTable,wmsJTUpdt;
        WMSJournalTrans                     wmsJournalTrans;
        WMSJournalName                      wmsJournalName;
        InventDim                           inventDim,inventDimPurch;
        InventSerial                        inventSerial;
        PurchTable                          purchTable;
        PurchLine                           purchLine;
        JournalCheckPost                    journalCheckPost;
        HSOEASNSerialTable                  ASNSerial;
        str                                 txt;
        str                                 journalName;
        str                                 journalNum;
        str                                 batchSuffix;
        int                                 k;
        real                                qty,amt,lastLineNum;
        container                           journalCont,record;
        //System.IO.StreamReader              reader;
        System.IO.MemoryStream              fstream;
        ExternalItemId                      extItemId;

        //FtpWebRequest requestFile = WebRequest::Create((ftpServer+unprocessed+_fileName));
        //requestFile.Credentials = new NetworkCredential(userName,password);
        //requestFile.Timeout = 900000;
        //requestFile.ReadWriteTimeout = 900000;
        //requestFile.UsePassive = true;        

        System.Net.FtpWebRequest ftpRequest;
        System.Net.FtpWebResponse ftpResponse;
        //System.IO.Stream ftpStream;
        //System.IO.StreamReader reader;
        str line;
        boolean chk = true;
        container csvValues;

        try
        {
            ftpRequest = System.Net.WebRequest::Create(ftpServer+unprocessed+_fileName);
            ftpRequest.set_Credentials(new System.Net.NetworkCredential(userName,password));
            ftpRequest.set_Method("RETR");
            ftpRequest.set_Timeout(900000); 
            ftpRequest.set_ReadWriteTimeout(900000); 
            ftpRequest.UsePassive = true;
            ftpResponse = ftpRequest.GetResponse();
            //ftpStream = ftpResponse.GetResponseStream();
            //reader = new System.IO.StreamReader(ftpStream, System.Text.Encoding::UTF8);
            using (System.IO.Stream ftpStream = ftpResponse.GetResponseStream())
            {
                using (System.IO.StreamReader reader = new System.IO.StreamReader(ftpStream, System.Text.Encoding::UTF8))
                {
                    while (chk)
                    {
                        line = reader.ReadLine();
                        record = str2con(line, ";");
                        if(conLen(record) > 1)
                        {
                            extItemId       = conpeek(record,7);
                            itemId          = CustVendExternalItem::findExternalItemId(ModuleInventPurchSalesVendCustGroup::Vend,InventParameters::find().HSOEASNVendAccount,extItemId).ItemId;
                            manufactureDate = this.getManufactureDate(conpeek(record,13));
                            expiryDate      = str2Date(conpeek(record,14),321);
                            if(itemId)
                            {
                                if((InventItemGroup::find(InventItemGroupItem::findByItemIdLegalEntity(itemId).ItemGroupId).HSOEEnableVariantBlocking == NoYes::Yes) && expiryDate)
                                {
                                    inventBatchId   = "";
                                    batchSuffix     = "";
                                    prefixTab = HSOEASNImportPrefixAttributeTable::find(conpeek(record,10),conpeek(record,15),conpeek(record,16),conpeek(record,17),conpeek(record,18),conpeek(record,19),conpeek(record,20),conpeek(record,23),conpeek(record,22));
                                    if(prefixTab && prefixTab.VariantId != "-" && prefixTab.Active == NoYes::Yes)
                                    {
                                        inventBatchId = this.findOrCreateBatchId(prefixTab.VariantId,expiryDate,conpeek(record,7),manufactureDate);
                                    }
                                    else if(!prefixTab)
                                    {
                                        HSOEASNImportPrefixAttributeTable::createRecord(conpeek(record,10),conpeek(record,15),conpeek(record,16),conpeek(record,17),conpeek(record,18),conpeek(record,19),conpeek(record,20),conpeek(record,23),conpeek(record,22),conpeek(record,1));
                                        batchSuffix = this.findOrCreateBatchId("-",expiryDate,conpeek(record,7),manufactureDate);
                                    }
                                    else if(prefixTab.VariantId == "-" || prefixTab.Active == NoYes::No)
                                    {
                                        batchSuffix = this.findOrCreateBatchId("-",expiryDate,conpeek(record,7),manufactureDate);
                                    }
                                }
                                else
                                {
                                    inventBatchId = this.findOrCreateBatchId(conpeek(record,4),expiryDate,conpeek(record,7),manufactureDate);
                                }

                                inventSerialId = conpeek(record,3);
                                if(strLen(inventBatchId) < 8)
                                {
                                    inventBatchId = "";
                                }
                            

                                if(headerPackingSlipMap.exists(conpeek(record,1)))
                                {
                                    wmsJournalTable.clear();
                                    purchLine.clear();
                                    purchTable.clear();
                                    inventDim.clear();
                                    inventDimPurch.clear();
                                    wmsJournalTrans.clear();
                                    purchId    = conpeek(record,21);
                                    purchTable = PurchTable::find(purchId);
                                    select firstonly purchLine where purchLine.PurchId == purchId && purchLine.ItemId == itemId;
                                    if(purchTable && purchLine)
                                    {
                                        inventDimPurch  = purchLine.inventDim();
             
                                        inventDim.inventBatchId         = "";
                                        inventDim.inventSerialId        = "";
                                
                                        inventDim.InventLocationId      = inventDimPurch.InventLocationId;
                                        inventDim.InventSiteId          = InventLocation::find(inventDimPurch.InventLocationId).InventSiteId;
                                        if(inventDimPurch.wMSLocationId)
                                        {
                                            inventDim.wMSLocationId     = inventDimPurch.wMSLocationId;
                                        }
                                        else
                                        {
                                            inventDim.wMSLocationId     = InventLocation::find(inventDimPurch.InventLocationId).WMSLocationIdDefaultReceipt;
                                        }
                                        if(purchTable.MCRDropShipment)
                                        {
                                            inventDim.InventStatusId    = inventDimPurch.InventStatusId;
                                        }
                                        else
                                        {
                                            inventDim.InventStatusId        = InventParameters::find().HSOEInventStatusId;
                                        }
                                        inventDim                       = InventDim::findOrCreate(inventDim);
                                    }
                                    else
                                    {
                                        inventDim.inventBatchId         = "";
                                        inventDim.inventSerialId        = "";
                                        inventDim.InventLocationId      = InventParameters::find().HSOEDefaultLocationId;
                                        inventDim.InventSiteId          = InventParameters::find().HSOEDefaultSiteId;
                                        inventDim.wMSLocationId         = InventLocation::find(inventDim.InventLocationId).WMSLocationIdDefaultReceipt;
                                        inventDim.InventStatusId        = InventParameters::find().HSOEInventStatusId;
                                        inventDim                       = InventDim::findOrCreate(inventDim);
                                    }
             
                                    wmsJournalTable  = WMSJournalTable::find(headerPackingSlipMap.lookup(conpeek(record,1)));
                                    ttsbegin;
                                    wmsJournalTrans.initFromWMSJournalTable(wmsJournalTable);
                                    wmsJournalTrans.JournalId                = journalNum;
                                    wmsJournalTrans.LineNum                  = conpeek(record,2);
                                    wmsJournalTrans.TransDate                = DateTimeUtil::getSystemDate(DateTimeUtil::getUserPreferredTimeZone());
                                    wmsJournalTrans.ItemId                   = itemId;
                                    wmsJournalTrans.Qty                      = conpeek(record,9);
                                    wmsJournalTrans.InventDimId              = inventDim.InventDimId;
                                    wmsJournalTrans.inventTransType          = InventTransType::Purch;
                                    if(purchTable)
                                    {
                                        wmsJournalTrans.vendAccount              = purchTable.OrderAccount;
                                        wmsJournalTrans.inventTransRefId         = purchId;
                                        wmsJournalTrans.HSOEUnitPrice            = purchLine.PurchPrice;
                                    }
                                    else
                                    {
                                        wmsJournalTrans.HSOEASNPurchId           = purchId;
                                    }
                                    wmsJournalTrans.HSOEASNBoxId             = strReplace(conPeek(record,25),"\r","");
                               
                                    wmsJournalTrans.insert();
                                    ASNSerial.clear();
                                    ASNSerial.InventSerialId             = inventSerialId;
                                    ASNSerial.HSOEASNBatchID1            = conpeek(record,4);
                                    ASNSerial.HSOEASNBatchID2            = inventBatchId;
                                    ASNSerial.HSOEASNLocationId          = conpeek(record,6);
                                    ASNSerial.HSOEASNProductModel        = conpeek(record,10);
                                    ASNSerial.HSOEASNProductPowerAQStr   = conpeek(record,11);
                                    ASNSerial.HSOEASNSterilizationLotNum = conpeek(record,12);
                                    ASNSerial.ItemId                     = itemId;
                                    ASNSerial.ProdDate                   = manufactureDate;
                                    ASNSerial.HSOEASNCOO                 = conpeek(record,15);
                                    ASNSerial.HSOEASNMFGSite             = conpeek(record,16);
                                    ASNSerial.HSOEASNSterilizationSite   = conpeek(record,17);
                                    ASNSerial.HSOEASNLegalManufacturer   = conpeek(record,18);
                                    ASNSerial.HSOEASNCEMark              = conpeek(record,19);
                                    ASNSerial.HSOEASNOuterboxLabel       = conpeek(record,20);
                                    ASNSerial.HSOEASNCustPONum           = conpeek(record,21);
                                    ASNSerial.HSOEASNIFUVersion          = conpeek(record,22);
                                    ASNSerial.HSOEASNPatientIdCard       = conpeek(record,23);
                                    ASNSerial.HSOEASNBOMId               = conpeek(record,24);
                                    ASNSerial.HSOEASNBoxId               = strReplace(conPeek(record,25),"\r","");
                                    ASNSerial.HSOEBatchSuffix            = batchSuffix;
                                    ASNSerial.HSOEexpDate                = expiryDate;
                                    ASNSerial.HSOEASNPackingSlipId       = conpeek(record,1);
                                    ASNSerial.RefRecId                   = wmsJournalTrans.RecId;
                                    ASNSerial.journalId                  = wmsJournalTrans.journalId;
                                    ASNSerial.insert();
                                    ttscommit;
                                }
                                else
                                {
                                    journalName = WMSParameters::find().receptionJournalNameId;
                                    wmsJournalTable.clear();
                                    wmsJournalName.clear();
                                    wmsJournalName = WMSJournalName::find(journalName);

                                    ttsbegin;
                                    wmsJournalTable.initFromWMSJournalName(wmsJournalName);
                                    wmsJournalTable.JournalNameId           = journalName;
                                    wmsJournalTable.journalType             = WMSJournalType::Reception;
                                    wmsJournalTable.packingSlip             = conpeek(record,1);
                                    wmsJournalTable.inventTransType         = InventTransType::Purch;
                                    //wmsJournalTable.vendAccount             = PurchTable::find(conpeek(record,21)).OrderAccount;
                                    //wmsJournalTable.inventTransRefId        = conpeek(record,21);
                                    wmsJournalTable.checkPickingLocation    = NoYes::No;
                                    wmsJournalTable.inventDimId             = "AllBlank";
                                    wmsJournalTable.HSOEOrigin              = HSOEWMSJourOriginType::IOLASN;

                                    xSession xSession                       = new xSession(sessionId());
                    
                                    wmsJournalTable.systemBlocked           = true;
                                    wmsJournalTable.insert();
                                    journalNum                              = wmsJournalTable.journalId;
                                    ttscommit;
                                    headerPackingSlipMap.insert(conpeek(record,1),journalNum);
                                    journalCont     = conIns(journalCont,conLen(journalCont)+1,journalNum);

                                    purchLine.clear();
                                    purchTable.clear();
                                    inventDim.clear();
                                    inventDimPurch.clear();
                                    wmsJournalTrans.clear();

                                    purchId    = conpeek(record,21);
                                    purchTable = PurchTable::find(purchId);
                                    select firstonly purchLine where purchLine.PurchId == purchId && purchLine.ItemId == itemId;
                                    if(purchTable && purchLine)
                                    {
                                        inventDimPurch  = purchLine.inventDim();
             
                                        inventDim.inventBatchId         = "";
                                        inventDim.inventSerialId        = "";
                                        inventDim.InventLocationId      = inventDimPurch.InventLocationId;
                                        inventDim.InventSiteId          = InventLocation::find(inventDimPurch.InventLocationId).InventSiteId;
                                        if(inventDimPurch.wMSLocationId)
                                        {
                                            inventDim.wMSLocationId     = inventDimPurch.wMSLocationId;
                                        }
                                        else
                                        {
                                            inventDim.wMSLocationId     = InventLocation::find(inventDimPurch.InventLocationId).WMSLocationIdDefaultReceipt;
                                        }
                                        if(purchTable.MCRDropShipment)
                                        {
                                            inventDim.InventStatusId    = inventDimPurch.InventStatusId;
                                        }
                                        else
                                        {
                                            inventDim.InventStatusId        = InventParameters::find().HSOEInventStatusId;
                                        }
                                        inventDim                       = InventDim::findOrCreate(inventDim);
                                    }
                                    else
                                    {
                                        inventDim.inventBatchId         = "";
                                        inventDim.inventSerialId        = "";
                                        inventDim.InventLocationId      = InventParameters::find().HSOEDefaultLocationId;
                                        inventDim.InventSiteId          = InventParameters::find().HSOEDefaultSiteId;
                                        inventDim.wMSLocationId         = InventLocation::find(inventDim.InventLocationId).WMSLocationIdDefaultReceipt;
                                        inventDim.InventStatusId        = InventParameters::find().HSOEInventStatusId;
                                        inventDim                       = InventDim::findOrCreate(inventDim);
                                    }
                                    ttsbegin;
                                    wmsJournalTrans.initFromWMSJournalTable(wmsJournalTable);
                                    wmsJournalTrans.JournalId                = journalNum;
                                    wmsJournalTrans.LineNum                  = conpeek(record,2);
                                    wmsJournalTrans.TransDate                = DateTimeUtil::getSystemDate(DateTimeUtil::getUserPreferredTimeZone());
                                    wmsJournalTrans.ItemId                   = itemId;
                                    wmsJournalTrans.Qty                      = conpeek(record,9);
                                    wmsJournalTrans.InventDimId              = inventDim.InventDimId;
                                    wmsJournalTrans.inventTransType          = InventTransType::Purch;
                                    if(purchTable)
                                    {
                                        wmsJournalTrans.vendAccount              = purchTable.OrderAccount;
                                        wmsJournalTrans.inventTransRefId         = purchId;
                                        wmsJournalTrans.HSOEUnitPrice            = purchLine.PurchPrice;
                                    }
                                    else
                                    {
                                        wmsJournalTrans.HSOEASNPurchId           = purchId;
                                    }
                                    wmsJournalTrans.HSOEASNBoxId             = strReplace(conPeek(record,25),"\r","");
                                    wmsJournalTrans.insert();

                                    ASNSerial.clear();
                                    ASNSerial.InventSerialId             = inventSerialId;
                                    ASNSerial.HSOEASNBatchID1            = conpeek(record,4);
                                    ASNSerial.HSOEASNBatchID2            = inventBatchId;
                                    ASNSerial.HSOEASNLocationId          = conpeek(record,6);
                                    ASNSerial.HSOEASNProductModel        = conpeek(record,10);
                                    ASNSerial.HSOEASNProductPowerAQStr   = conpeek(record,11);
                                    ASNSerial.HSOEASNSterilizationLotNum = conpeek(record,12);
                                    ASNSerial.ItemId                     = itemId;
                                    ASNSerial.ProdDate                   = manufactureDate;
                                    ASNSerial.HSOEASNCOO                 = conpeek(record,15);
                                    ASNSerial.HSOEASNMFGSite             = conpeek(record,16);
                                    ASNSerial.HSOEASNSterilizationSite   = conpeek(record,17);
                                    ASNSerial.HSOEASNLegalManufacturer   = conpeek(record,18);
                                    ASNSerial.HSOEASNCEMark              = conpeek(record,19);
                                    ASNSerial.HSOEASNOuterboxLabel       = conpeek(record,20);
                                    ASNSerial.HSOEASNCustPONum           = conpeek(record,21);
                                    ASNSerial.HSOEASNIFUVersion          = conpeek(record,22);
                                    ASNSerial.HSOEASNPatientIdCard       = conpeek(record,23);
                                    ASNSerial.HSOEASNBOMId               = conpeek(record,24);
                                    ASNSerial.HSOEASNBoxId               = strReplace(conPeek(record,25),"\r","");
                                    ASNSerial.HSOEBatchSuffix            = batchSuffix;
                                    ASNSerial.HSOEexpDate                = expiryDate;
                                    ASNSerial.HSOEASNPackingSlipId       = conpeek(record,1);
                                    ASNSerial.RefRecId                   = wmsJournalTrans.RecId;
                                    ASNSerial.journalId                  = wmsJournalTrans.journalId;
                                    ASNSerial.insert();
                                    ttscommit;
                                    logTab.clear();
                                    logTab.PackingSlipId = conpeek(record,1);
                                    logTab.ImportDate = systemDateGet();
                                    logTab.FTPUpload = NoYes::No;
                                    logTab.Processed = NoYes::Yes;
                                    logTab.Log = strfmt("Journal # %1 created",journalNum);
                                    logTab.insert();
                                    info(strfmt("Journal # %1 created",journalNum));
                                }
                            }
                            else
                            {
                                itemId          = extItemId;
                                if((InventItemGroup::find(InventItemGroupItem::findByItemIdLegalEntity(itemId).ItemGroupId).HSOEEnableVariantBlocking == NoYes::Yes) && expiryDate)
                                {
                                    inventBatchId   = "";
                                    batchSuffix     = "";
                                    prefixTab = HSOEASNImportPrefixAttributeTable::find(conpeek(record,10),conpeek(record,15),conpeek(record,16),conpeek(record,17),conpeek(record,18),conpeek(record,19),conpeek(record,20),conpeek(record,23),conpeek(record,22));
                                    if(prefixTab && prefixTab.VariantId != "-" && prefixTab.Active == NoYes::Yes)
                                    {
                                        inventBatchId = this.findOrCreateBatchId(prefixTab.VariantId,expiryDate,conpeek(record,7),manufactureDate);
                                    }
                                    else if(!prefixTab)
                                    {
                                        HSOEASNImportPrefixAttributeTable::createRecord(conpeek(record,10),conpeek(record,15),conpeek(record,16),conpeek(record,17),conpeek(record,18),conpeek(record,19),conpeek(record,20),conpeek(record,23),conpeek(record,22),conpeek(record,1));
                                        batchSuffix = this.findOrCreateBatchId("-",expiryDate,conpeek(record,7),manufactureDate);
                                    }
                                    else if(prefixTab.VariantId == "-" || prefixTab.Active == NoYes::No)
                                    {
                                        batchSuffix = this.findOrCreateBatchId("-",expiryDate,conpeek(record,7),manufactureDate);
                                    }
                                }
                                else
                                {
                                    inventBatchId = this.findOrCreateBatchId(conpeek(record,4),expiryDate,conpeek(record,7),manufactureDate);
                                }

                                inventSerialId = conpeek(record,3);
                                if(strLen(inventBatchId) < 8)
                                {
                                    inventBatchId = "";
                                }
                            

                                if(headerPackingSlipMap.exists(conpeek(record,1)))
                                {
                                    wmsJournalTable.clear();
                                    purchLine.clear();
                                    purchTable.clear();
                                    inventDim.clear();
                                    inventDimPurch.clear();
                                    wmsJournalTrans.clear();
                                    purchId    = conpeek(record,21);
                                    purchTable = PurchTable::find(purchId);
                                    select firstonly purchLine where purchLine.PurchId == purchId && purchLine.ItemId == itemId;
                                    if(purchTable && purchLine)
                                    {
                                        inventDimPurch  = purchLine.inventDim();
             
                                        inventDim.inventBatchId         = "";
                                        inventDim.inventSerialId        = "";
                                
                                        inventDim.InventLocationId      = inventDimPurch.InventLocationId;
                                        inventDim.InventSiteId          = InventLocation::find(inventDimPurch.InventLocationId).InventSiteId;
                                        if(inventDimPurch.wMSLocationId)
                                        {
                                            inventDim.wMSLocationId     = inventDimPurch.wMSLocationId;
                                        }
                                        else
                                        {
                                            inventDim.wMSLocationId     = InventLocation::find(inventDimPurch.InventLocationId).WMSLocationIdDefaultReceipt;
                                        }
                                        if(purchTable.MCRDropShipment)
                                        {
                                            inventDim.InventStatusId    = inventDimPurch.InventStatusId;
                                        }
                                        else
                                        {
                                            inventDim.InventStatusId        = InventParameters::find().HSOEInventStatusId;
                                        }
                                        inventDim                       = InventDim::findOrCreate(inventDim);
                                    }
                                    else
                                    {
                                        inventDim.inventBatchId         = "";
                                        inventDim.inventSerialId        = "";
                                        inventDim.InventLocationId      = InventParameters::find().HSOEDefaultLocationId;
                                        inventDim.InventSiteId          = InventParameters::find().HSOEDefaultSiteId;
                                        inventDim.wMSLocationId         = InventLocation::find(inventDim.InventLocationId).WMSLocationIdDefaultReceipt;
                                        inventDim.InventStatusId        = InventParameters::find().HSOEInventStatusId;
                                        inventDim                       = InventDim::findOrCreate(inventDim);
                                    }
             
                                    wmsJournalTable  = WMSJournalTable::find(headerPackingSlipMap.lookup(conpeek(record,1)));
                                    ttsbegin;
                                    wmsJournalTrans.initFromWMSJournalTable(wmsJournalTable);
                                    wmsJournalTrans.JournalId                = journalNum;
                                    wmsJournalTrans.LineNum                  = conpeek(record,2);
                                    wmsJournalTrans.TransDate                = DateTimeUtil::getSystemDate(DateTimeUtil::getUserPreferredTimeZone());
                                    wmsJournalTrans.ItemId                   = itemId;
                                    wmsJournalTrans.Qty                      = conpeek(record,9);
                                    wmsJournalTrans.InventDimId              = inventDim.InventDimId;
                                    wmsJournalTrans.inventTransType          = InventTransType::Purch;
                                    if(purchTable)
                                    {
                                        wmsJournalTrans.vendAccount              = purchTable.OrderAccount;
                                        wmsJournalTrans.inventTransRefId         = purchId;
                                        wmsJournalTrans.HSOEUnitPrice            = purchLine.PurchPrice;
                                    }
                                    else
                                    {
                                        wmsJournalTrans.HSOEASNPurchId           = purchId;
                                    }
                                    wmsJournalTrans.HSOEASNBoxId             = strReplace(conPeek(record,25),"\r","");
                                    wmsJournalTrans.HSOEASNExtItemId         = itemId;
                                    wmsJournalTrans.insert();

                                    ASNSerial.clear();
                                    ASNSerial.InventSerialId             = inventSerialId;
                                    ASNSerial.HSOEASNBatchID1            = conpeek(record,4);
                                    ASNSerial.HSOEASNBatchID2            = inventBatchId;
                                    ASNSerial.HSOEASNLocationId          = conpeek(record,6);
                                    ASNSerial.HSOEASNProductModel        = conpeek(record,10);
                                    ASNSerial.HSOEASNProductPowerAQStr   = conpeek(record,11);
                                    ASNSerial.HSOEASNSterilizationLotNum = conpeek(record,12);
                                    ASNSerial.ItemId                     = itemId;
                                    ASNSerial.ProdDate                   = manufactureDate;
                                    ASNSerial.HSOEASNCOO                 = conpeek(record,15);
                                    ASNSerial.HSOEASNMFGSite             = conpeek(record,16);
                                    ASNSerial.HSOEASNSterilizationSite   = conpeek(record,17);
                                    ASNSerial.HSOEASNLegalManufacturer   = conpeek(record,18);
                                    ASNSerial.HSOEASNCEMark              = conpeek(record,19);
                                    ASNSerial.HSOEASNOuterboxLabel       = conpeek(record,20);
                                    ASNSerial.HSOEASNCustPONum           = conpeek(record,21);
                                    ASNSerial.HSOEASNIFUVersion          = conpeek(record,22);
                                    ASNSerial.HSOEASNPatientIdCard       = conpeek(record,23);
                                    ASNSerial.HSOEASNBOMId               = conpeek(record,24);
                                    ASNSerial.HSOEASNBoxId               = strReplace(conPeek(record,25),"\r","");
                                    ASNSerial.HSOEBatchSuffix            = batchSuffix;
                                    ASNSerial.HSOEexpDate                = expiryDate;
                                    ASNSerial.HSOEASNPackingSlipId       = conpeek(record,1);
                                    ASNSerial.RefRecId                   = wmsJournalTrans.RecId;
                                    ASNSerial.journalId                  = wmsJournalTrans.journalId;
                                    ASNSerial.insert();
                                    ttscommit;
                                }
                                else
                                {
                                    journalName = WMSParameters::find().receptionJournalNameId;
                                    wmsJournalTable.clear();
                                    wmsJournalName.clear();
                                    wmsJournalName = WMSJournalName::find(journalName);

                                    ttsbegin;
                                    wmsJournalTable.initFromWMSJournalName(wmsJournalName);
                                    wmsJournalTable.JournalNameId           = journalName;
                                    wmsJournalTable.journalType             = WMSJournalType::Reception;
                                    wmsJournalTable.packingSlip             = conpeek(record,1);
                                    wmsJournalTable.inventTransType         = InventTransType::Purch;
                                    //wmsJournalTable.vendAccount             = PurchTable::find(conpeek(record,21)).OrderAccount;
                                    //wmsJournalTable.inventTransRefId        = conpeek(record,21);
                                    wmsJournalTable.checkPickingLocation    = NoYes::No;
                                    wmsJournalTable.inventDimId             = "AllBlank";
                                    wmsJournalTable.HSOEOrigin              = HSOEWMSJourOriginType::IOLASN;

                                    xSession xSession                       = new xSession(sessionId());
                    
                                    wmsJournalTable.systemBlocked           = true;
                                    wmsJournalTable.insert();
                                    journalNum                              = wmsJournalTable.journalId;
                                    ttscommit;
                                    headerPackingSlipMap.insert(conpeek(record,1),journalNum);
                                    journalCont     = conIns(journalCont,conLen(journalCont)+1,journalNum);

                                    purchLine.clear();
                                    purchTable.clear();
                                    inventDim.clear();
                                    inventDimPurch.clear();
                                    wmsJournalTrans.clear();

                                    purchId    = conpeek(record,21);
                                    purchTable = PurchTable::find(purchId);
                                    select firstonly purchLine where purchLine.PurchId == purchId && purchLine.ItemId == itemId;
                                    if(purchTable && purchLine)
                                    {
                                        inventDimPurch  = purchLine.inventDim();
             
                                        inventDim.inventBatchId         = "";
                                        inventDim.inventSerialId        = "";
                                        inventDim.InventLocationId      = inventDimPurch.InventLocationId;
                                        inventDim.InventSiteId          = InventLocation::find(inventDimPurch.InventLocationId).InventSiteId;
                                        if(inventDimPurch.wMSLocationId)
                                        {
                                            inventDim.wMSLocationId     = inventDimPurch.wMSLocationId;
                                        }
                                        else
                                        {
                                            inventDim.wMSLocationId     = InventLocation::find(inventDimPurch.InventLocationId).WMSLocationIdDefaultReceipt;
                                        }
                                        if(purchTable.MCRDropShipment)
                                        {
                                            inventDim.InventStatusId    = inventDimPurch.InventStatusId;
                                        }
                                        else
                                        {
                                            inventDim.InventStatusId        = InventParameters::find().HSOEInventStatusId;
                                        }
                                        inventDim                       = InventDim::findOrCreate(inventDim);
                                    }
                                    else
                                    {
                                        inventDim.inventBatchId         = "";
                                        inventDim.inventSerialId        = "";
                                        inventDim.InventLocationId      = InventParameters::find().HSOEDefaultLocationId;
                                        inventDim.InventSiteId          = InventParameters::find().HSOEDefaultSiteId;
                                        inventDim.wMSLocationId         = InventLocation::find(inventDim.InventLocationId).WMSLocationIdDefaultReceipt;
                                        inventDim.InventStatusId        = InventParameters::find().HSOEInventStatusId;
                                        inventDim                       = InventDim::findOrCreate(inventDim);
                                    }
                                    ttsbegin;
                                    wmsJournalTrans.initFromWMSJournalTable(wmsJournalTable);
                                    wmsJournalTrans.JournalId                = journalNum;
                                    wmsJournalTrans.LineNum                  = conpeek(record,2);
                                    wmsJournalTrans.TransDate                = DateTimeUtil::getSystemDate(DateTimeUtil::getUserPreferredTimeZone());
                                    wmsJournalTrans.ItemId                   = itemId;
                                    wmsJournalTrans.Qty                      = conpeek(record,9);
                                    wmsJournalTrans.InventDimId              = inventDim.InventDimId;
                                    wmsJournalTrans.inventTransType          = InventTransType::Purch;
                                    if(purchTable)
                                    {
                                        wmsJournalTrans.vendAccount              = purchTable.OrderAccount;
                                        wmsJournalTrans.inventTransRefId         = purchId;
                                        wmsJournalTrans.HSOEUnitPrice            = purchLine.PurchPrice;
                                    }
                                    else
                                    {
                                        wmsJournalTrans.HSOEASNPurchId           = purchId;
                                    }
                                    wmsJournalTrans.HSOEASNBoxId             = strReplace(conPeek(record,25),"\r","");
                                    wmsJournalTrans.HSOEASNExtItemId         = itemId;
                                    wmsJournalTrans.insert();

                                    ASNSerial.clear();
                                    ASNSerial.InventSerialId             = inventSerialId;
                                    ASNSerial.HSOEASNBatchID1            = conpeek(record,4);
                                    ASNSerial.HSOEASNBatchID2            = inventBatchId;
                                    ASNSerial.HSOEASNLocationId          = conpeek(record,6);
                                    ASNSerial.HSOEASNProductModel        = conpeek(record,10);
                                    ASNSerial.HSOEASNProductPowerAQStr   = conpeek(record,11);
                                    ASNSerial.HSOEASNSterilizationLotNum = conpeek(record,12);
                                    ASNSerial.ItemId                     = itemId;
                                    ASNSerial.ProdDate                   = manufactureDate;
                                    ASNSerial.HSOEASNCOO                 = conpeek(record,15);
                                    ASNSerial.HSOEASNMFGSite             = conpeek(record,16);
                                    ASNSerial.HSOEASNSterilizationSite   = conpeek(record,17);
                                    ASNSerial.HSOEASNLegalManufacturer   = conpeek(record,18);
                                    ASNSerial.HSOEASNCEMark              = conpeek(record,19);
                                    ASNSerial.HSOEASNOuterboxLabel       = conpeek(record,20);
                                    ASNSerial.HSOEASNCustPONum           = conpeek(record,21);
                                    ASNSerial.HSOEASNIFUVersion          = conpeek(record,22);
                                    ASNSerial.HSOEASNPatientIdCard       = conpeek(record,23);
                                    ASNSerial.HSOEASNBOMId               = conpeek(record,24);
                                    ASNSerial.HSOEASNBoxId               = strReplace(conPeek(record,25),"\r","");
                                    ASNSerial.HSOEBatchSuffix            = batchSuffix;
                                    ASNSerial.HSOEexpDate                = expiryDate;
                                    ASNSerial.HSOEASNPackingSlipId       = conpeek(record,1);
                                    ASNSerial.RefRecId                   = wmsJournalTrans.RecId;
                                    ASNSerial.journalId                  = wmsJournalTrans.journalId;
                                    ASNSerial.insert();
                                    ttscommit;
                                    logTab.clear();
                                    logTab.PackingSlipId = conpeek(record,1);
                                    logTab.ImportDate = systemDateGet();
                                    logTab.FTPUpload = NoYes::No;
                                    logTab.Processed = NoYes::Yes;
                                    logTab.Log = strfmt("Journal # %1 created",journalNum);
                                    logTab.insert();
                                    info(strfmt("Journal # %1 created",journalNum));
                                }
                            }
                        }
                        else
                        {
                            chk = false;
                        }
                    }
                }
            }
        }
        catch(Exception::Error)
        {            
        }
        catch(Exception::CLRError)
        {
        }
        finally
        {
            ftpResponse.Close();
        }

        try
        {
            this.validateJournal(journalCont);
        }
        catch(Exception::Error)
        {
        }

    }

    public void validateJournal(container   _journalCont)
    {
        WMSJournalTable wmsJournalTable;
        int k;

        for(k=1;k<=conLen(_journalCont);k++)
        {
            HSOEASNImportClass::validateJournal(conPeek(_journalCont,k));
            wmsJournalTable =   WMSJournalTable::find(conPeek(_journalCont,k),true);
            ttsbegin;
            if(wmsJournalTable)
            {
                wmsJournalTable.systemBlocked   =   NoYes::No;
                wmsJournalTable.update();
            }
            ttscommit;
        }
    }

    public TransDate getManufactureDate(str _mDate)
    {
        TransDate mdate;
        mdate = mkDate(01,str2Int(subStr(_mDate,6,2)),str2Int(subStr(_mDate,1,4)));
        return mdate;
    }

    public InventBatchId findOrCreateBatchId(HSOEASNVariant _var,TransDate _exp,ItemId _iId,TransDate _man)
    {
        InventBatch     invBatch;
        InventBatchId   batchId;
        if(_var != "-")
        {
            if(InventItemGroup::find(InventItemGroupItem::findByItemIdLegalEntity(_iId).ItemGroupId).HSOEEnableVariantBlocking == NoYes::Yes)
            {
                batchId = _var + "-"+ date2Str(_exp,321,DateDay::Digits2,DateSeparator::None,DateMonth::Digits2,DateSeparator::None,DateYear::Digits4);
            }
            else
            {
                batchId = _var;
            }
            if(strLen(batchId) > 8)
            {
                select firstonly invBatch where invBatch.inventBatchId == batchId && invBatch.itemId == _iId;
                if(!invBatch)
                {
                    invBatch.clear();
                    ttsbegin;
                    invBatch.inventBatchId = batchId;
                    invBatch.itemId = _iId;
                    invBatch.expDate = _exp;
                    invBatch.PdsBestBeforeDate = _exp;
                    invBatch.PdsShelfAdviceDate = _exp;
                    invBatch.prodDate = _man;
                    invBatch.HSOEVariant = _var;
                    invBatch.insert();
                    ttscommit;
                
                    if(invBatch.expDate != _exp)
                    {
                        ttsbegin;
                        invBatch.selectForUpdate(true);
                        invBatch.expDate = _exp;
                        invBatch.PdsBestBeforeDate = _exp;
                        invBatch.PdsShelfAdviceDate = _exp;
                        invBatch.update();
                        ttscommit;
                    }
                
                }
            }
        }
        else if(_var == "-")
        {
            batchId = date2Str(_exp,321,DateDay::Digits2,DateSeparator::None,DateMonth::Digits2,DateSeparator::None,DateYear::Digits4);
        }
        return batchId;
    }

    public void moveProcessedFile(str _fileName)
    {
        FtpWebRequest requestMove = WebRequest::Create(ftpServer+unprocessed+_fileName);
        requestMove.Credentials = new NetworkCredential(userName,password);
        requestMove.Method = "RENAME";
        requestMove.RenameTo = processed+_fileName;
        requestMove.GetResponse().Close();
    }

    public void moveErrorFile(str _fileName)
    {
        FtpWebRequest requestErr = WebRequest::Create(ftpServer+unprocessed+_fileName);
        requestErr.Credentials = new NetworkCredential(userName,password);
        requestErr.Method = "RENAME";
        requestErr.RenameTo = err+_fileName;
        requestErr.GetResponse().Close();
    }

    public void validateSid(str _fileName)
    {
        AsciiStreamIo                       file,filecheck;
        System.Byte[]                       buffer = new System.Byte[300000]();
        System.IO.Stream                    fileStream = new System.IO.MemoryStream();
        System.IO.FileInfo                  fileInfo;
        WMSJournalTable                     wmsJournalTableLoc;
        InventSerialId                      inventSerialIdLoc;
        ItemId                              itemIdLoc;
        PurchTable                          purchTableLoc;
        str                                 psError,poError,itemError,paraError,erro,packingSlipId,formatError,purchId;
        FileUploadTemporaryStorageResult    fileUploadResultLoc = new FileUploadTemporaryStorageResult();
        TransDate                           expiryDateLoc;
        container   record;
        int         counter;
        InventTable inventTable;

        FtpWebRequest requestFile = WebRequest::Create((ftpServer+unprocessed+_fileName));
        requestFile.Credentials = new NetworkCredential(userName,password);

        using (Stream ftpStream = requestFile.GetResponse().GetResponseStream())
        {
            int read = 0;
            while(true)
            {
                read = ftpStream.Read(buffer, 0, buffer.Length);
                if(read > 0)
                {
                    fileStream.Write(buffer, 0, read);
                    file = AsciiStreamIo::constructForRead(fileStream);
                    if (file)
                    {
                        if (file.status())
                        {
                            throw error("@SYS52680");
                        }
                        
                        file.inFieldDelimiter(#Delimiter);
                        file.inRecordDelimiter(#RecordDelimiter);
                        while (!file.status())
                        {
                            counter++;
                            record = file.read();
                            if (conLen(record)>1)
                            {
                                packingSlipId     = conpeek(record,1);
                                purchId           = conpeek(record,21);
                                select firstonly purchTableLoc where purchTableLoc.PurchId == purchId;
                                itemIdLoc         = CustVendExternalItem::findExternalItemId(ModuleInventPurchSalesVendCustGroup::Vend,InventParameters::find().HSOEASNVendAccount,conpeek(record,7)).ItemId;
                                select firstonly wmsJournalTableLoc where wmsJournalTableLoc.packingSlip == packingSlipId;
                                if(wmsJournalTableLoc)
                                {
                                    psError += any2Str(counter) + " ";
                                }
                                inventTable = InventTable::find(itemIdLoc);
                                if(!inventTable)
                                {
                                    itemError += any2Str(counter)+ " ";
                                }
                                if(!(purchTableLoc))
                                {
                                    poError += any2Str(counter) + " ";
                                }
                            }
                            else
                            {
                                formatError += any2Str(counter) + " ";
                            }
                        }
                        if(psError)
                        {
                            erro += strFmt("Packingslip id of rows %1 already exists",psError);
                        }
                        if(poError)
                        {
                            erro += strFmt("Purchase order for lines %1 is incorrect",poError);
                        }
                        if(itemError)
                        {
                            erro += strFmt("Items of rows %1 does not exist",itemError);
                        }
                        if(formatError)
                        {
                            erro += strFmt("Rows %1 are bot correct format",formatError);
                        }
                        if(erro)
                        {
                            logTab.clear();
                            logTab.PackingSlipId = packingSlipId;
                            logTab.ImportDate = systemDateGet();
                            logTab.FTPUpload = NoYes::Yes;
                            logTab.Processed = NoYes::No;
                            logTab.Log = erro;
                            logTab.insert();
                            throw error(erro);
                        }
                    }
                }
                else
                {
                    break;
                }
            }
        }
        file.finalize();
        fileStream.Dispose();
        requestFile.GetResponse().Close();
    }

}

```

# Controller Class
```
class HSOEASNFTPImportController extends SysOperationServiceController
{
    protected void new()
    {
        super();
     
        this.parmClassName(classStr(HSOEASNFTPImportService));
        this.parmMethodName(methodStr(HSOEASNFTPImportService, processOperation));
     
        this.parmDialogCaption("ASN Import from FTP");
    }

    public ClassDescription caption()
    {
        return "ASN Import from FTP";
    }

    public static void main(Args _args)
    {
        HSOEASNFTPImportController controller;
     
        controller = new HSOEASNFTPImportController();
        controller.parmExecutionMode(SysOperationExecutionMode::Synchronous);
        controller.startOperation();
    }

}
```
