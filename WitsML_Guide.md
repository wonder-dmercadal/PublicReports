## Safe WITSML Querying from Command Prompt - Step by Step Guide

I'll guide you through **READ-ONLY** queries that are completely safe. These operations only retrieve data and cannot modify anything on the server.

### Prerequisites
1. **Ensure curl is installed** (Windows 10+ has it by default)
   ```cmd
   curl --version
   ```
   If not installed, you can download it from https://curl.se/windows/

2. **Create a working directory** for saving responses:
   ```cmd
   mkdir C:\WITSML_Queries
   cd C:\WITSML_Queries
   ```

### SAFE OPERATIONS ONLY
We'll only use these READ-ONLY operations:
- ✅ **WMLS_GetVersion** - Get server version
- ✅ **WMLS_GetCap** - Get server capabilities  
- ✅ **WMLS_GetFromStore** - Read data from server
- ❌ **Never use**: AddToStore, UpdateInStore, DeleteFromStore (these modify data)

---

## Step 1: Get Server Version (Safest First Query)

This is the simplest and safest query - just returns the WITSML version:

```cmd
curl -X POST http://witsml2.welldata.net/witsml/wmls.asmx ^
  -H "Content-Type: text/xml; charset=utf-8" ^
  -H "SOAPAction: \"http://www.witsml.org/action/120/Store.WMLS_GetVersion\"" ^
  -d "<?xml version=\"1.0\" encoding=\"utf-8\"?><soap:Envelope xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:tns=\"http://www.nov.com/witsml/\" xmlns:types=\"http://www.nov.com/witsml/encodedTypes\" xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\"><soap:Body soap:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\"><q1:WMLS_GetVersion xmlns:q1=\"http://www.witsml.org/message/120\"></q1:WMLS_GetVersion></soap:Body></soap:Envelope>" ^
  -o version_response.xml
```

**Note**: The `^` character is for line continuation in Windows Command Prompt. If using PowerShell, use `` ` `` instead.

---

## Step 2: Get Server Capabilities

This shows what data types are available:

```cmd
curl -X POST http://witsml2.welldata.net/witsml/wmls.asmx ^
  -H "Content-Type: text/xml; charset=utf-8" ^
  -H "SOAPAction: \"http://www.witsml.org/action/120/Store.WMLS_GetCap\"" ^
  -d "<?xml version=\"1.0\" encoding=\"utf-8\"?><soap:Envelope xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:tns=\"http://www.nov.com/witsml/\" xmlns:types=\"http://www.nov.com/witsml/encodedTypes\" xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\"><soap:Body soap:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\"><q1:WMLS_GetCap xmlns:q1=\"http://www.witsml.org/message/120\"><OptionsIn xsi:type=\"xsd:string\"></OptionsIn></q1:WMLS_GetCap></soap:Body></soap:Envelope>" ^
  -o capabilities_response.xml
```

---

## Step 3: Create Query Files (Safer Approach)

Instead of typing long commands, let's create XML files for our queries:

### 3a. Create a file for querying wells
Create a file named `get_wells.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
               xmlns:xsd="http://www.w3.org/2001/XMLSchema" 
               xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <q1:WMLS_GetFromStore xmlns:q1="http://www.witsml.org/message/120">
      <WMLtypeIn xsi:type="xsd:string">well</WMLtypeIn>
      <QueryIn xsi:type="xsd:string"><![CDATA[
        <wells xmlns="http://www.witsml.org/schemas/1series" version="1.4.1.1">
          <well uid="">
            <name/>
          </well>
        </wells>
      ]]></QueryIn>
      <OptionsIn xsi:type="xsd:string">returnElements=id-only</OptionsIn>
      <CapabilitiesIn xsi:type="xsd:string"></CapabilitiesIn>
    </q1:WMLS_GetFromStore>
  </soap:Body>
</soap:Envelope>
```

Run the query:
```cmd
curl -X POST http://witsml2.welldata.net/witsml/wmls.asmx ^
  -H "Content-Type: text/xml; charset=utf-8" ^
  -H "SOAPAction: \"http://www.witsml.org/action/120/Store.WMLS_GetFromStore\"" ^
  --data-binary @get_wells.xml ^
  -o wells_response.xml
```

---

## Step 4: Query for Logs with Equipment Data

### 4a. Create `get_logs.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
               xmlns:xsd="http://www.w3.org/2001/XMLSchema" 
               xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <q1:WMLS_GetFromStore xmlns:q1="http://www.witsml.org/message/120">
      <WMLtypeIn xsi:type="xsd:string">log</WMLtypeIn>
      <QueryIn xsi:type="xsd:string"><![CDATA[
        <logs xmlns="http://www.witsml.org/schemas/1series" version="1.4.1.1">
          <log uid="">
            <name/>
            <indexType/>
            <startIndex/>
            <endIndex/>
            <logCurveInfo>
              <mnemonic/>
              <classWitsml/>
              <unit/>
              <curveDescription/>
            </logCurveInfo>
          </log>
        </logs>
      ]]></QueryIn>
      <OptionsIn xsi:type="xsd:string">returnElements=header-only</OptionsIn>
      <CapabilitiesIn xsi:type="xsd:string"></CapabilitiesIn>
    </q1:WMLS_GetFromStore>
  </soap:Body>
</soap:Envelope>
```

Run:
```cmd
curl -X POST http://witsml2.welldata.net/witsml/wmls.asmx ^
  -H "Content-Type: text/xml; charset=utf-8" ^
  -H "SOAPAction: \"http://www.witsml.org/action/120/Store.WMLS_GetFromStore\"" ^
  --data-binary @get_logs.xml ^
  -o logs_response.xml
```

---

## Step 5: Search for Equipment-Specific Tags

### 5a. Create `get_equipment_curves.xml` to search for equipment data:

```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
               xmlns:xsd="http://www.w3.org/2001/XMLSchema" 
               xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <q1:WMLS_GetFromStore xmlns:q1="http://www.witsml.org/message/120">
      <WMLtypeIn xsi:type="xsd:string">log</WMLtypeIn>
      <QueryIn xsi:type="xsd:string"><![CDATA[
        <logs xmlns="http://www.witsml.org/schemas/1series" version="1.4.1.1">
          <log uid="">
            <logCurveInfo>
              <mnemonic>TD*</mnemonic>
            </logCurveInfo>
            <logCurveInfo>
              <mnemonic>VIB*</mnemonic>
            </logCurveInfo>
            <logCurveInfo>
              <mnemonic>*TEMP*</mnemonic>
            </logCurveInfo>
          </log>
        </logs>
      ]]></QueryIn>
      <OptionsIn xsi:type="xsd:string">returnElements=header-only</OptionsIn>
      <CapabilitiesIn xsi:type="xsd:string"></CapabilitiesIn>
    </q1:WMLS_GetFromStore>
  </soap:Body>
</soap:Envelope>
```

---

## Safety Best Practices

### 1. **Always Use Read-Only Operations**
- ✅ WMLS_GetFromStore (READ)
- ✅ WMLS_GetCap (READ)
- ✅ WMLS_GetVersion (READ)
- ❌ Never use: AddToStore, UpdateInStore, DeleteFromStore

### 2. **Use Query Options for Safety**
- `returnElements=id-only` - Returns only identifiers
- `returnElements=header-only` - Returns only metadata, no data values
- `maxReturnNodes=10` - Limits the amount of data returned

### 3. **Test Queries Incrementally**
1. Start with simple queries (GetVersion)
2. Query for object lists (wells, logs)
3. Then query for specific data

### 4. **Save All Responses**
Always use `-o filename.xml` to save responses for review

### 5. **Authentication (if required)**
If the server requires authentication, add:
```cmd
--user username:password
```

### 6. **View XML Responses Safely**
```cmd
type wells_response.xml | more
```

Or open in a text editor:
```cmd
notepad wells_response.xml
```

---

## Parsing Equipment Data

Once you find logs with equipment curves, you can query specific data. For example, if you find a log with TDTEMP (Top Drive Temperature):

Create `get_tdtemp_data.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
               xmlns:xsd="http://www.w3.org/2001/XMLSchema" 
               xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <q1:WMLS_GetFromStore xmlns:q1="http://www.witsml.org/message/120">
      <WMLtypeIn xsi:type="xsd:string">log</WMLtypeIn>
      <QueryIn xsi:type="xsd:string"><![CDATA[
        <logs xmlns="http://www.witsml.org/schemas/1series" version="1.4.1.1">
          <log uidWell="[WELL_UID]" uidWellbore="[WELLBORE_UID]" uid="[LOG_UID]">
            <logCurveInfo>
              <mnemonic>TDTEMP</mnemonic>
            </logCurveInfo>
            <startIndex uom="m">1000</startIndex>
            <endIndex uom="m">1100</endIndex>
          </log>
        </logs>
      ]]></QueryIn>
      <OptionsIn xsi:type="xsd:string">returnElements=data-only;maxReturnNodes=100</OptionsIn>
      <CapabilitiesIn xsi:type="xsd:string"></CapabilitiesIn>
    </q1:WMLS_GetFromStore>
  </soap:Body>
</soap:Envelope>
```

**Important**: Replace `[WELL_UID]`, `[WELLBORE_UID]`, and `[LOG_UID]` with actual values from previous queries.

---

## Troubleshooting

1. **If you get connection errors**: Check VPN connection
2. **If you get authentication errors**: You may need credentials
3. **If responses are too large**: Use `maxReturnNodes` option
4. **If you get encoding errors**: Ensure UTF-8 encoding in your files

Remember: These queries are completely safe as they only READ data. They cannot modify, delete, or add any data to the WITSML server.
