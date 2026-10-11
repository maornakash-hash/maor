# Howden Napa (Meitav Dash) – Daily report trigger: Notch code prompt

Paste the text below into the trigger's code-generation prompt.

```
Write Notch code for this trigger. It runs daily at 06:00 Asia/Jerusalem on the Excel daily report (V_Daily_Reports) and calls @Novidea (Salesforce) to fetch the matching offer. The code must output ONE JSON object per processed row ("transfer record"), and the shape must match the schema below exactly: same keys, same nesting, same types. Every key must always be present. Use null for missing or empty values, never "" and never omit a key. Key order does not matter.

## 1. Input and row change detection
- Load today's report rows and yesterday's report rows (the previous run's snapshot).
- Treat a cell as empty when it is blank, whitespace, "NULL", or the empty-date value "00:00.0".
- Row identity (rowKey): join the identity columns with "|", in this order: Mis_Zehut | Sug_Mutzar | Yatzran | Mispar_Polisa_From. Trim every value and drop empty values. Example: "23538937|קופת גמל|מיטב דש" (no transferring policy, so only 3 parts).
- Compare today against yesterday by rowKey:
  - In today only → rowChange = "new"
  - In yesterday only → rowChange = "deleted"
  - In both, and any column value differs → rowChange = "changed"
  - In both and identical → rowChange = "unchanged". Do NOT emit a record for these (Automation rule: act only on new, deleted and changed rows).
- For "deleted" rows, use the yesterday row everywhere this prompt says "the row".

## 2. Scope
A row is in scope only when ALL of these hold:
- Yatzran = "מיטב דש"
- Subject is "הצטרפויות עם ניוד" or "הצטרפויות ללא ניוד"
- Sug_Mutzar is "קופת גמל" or "קרן השתלמות" (גמל להשקעה is out of scope)
In scope → inScope = true, inScopeReason = "Sug_Mutzar=<value>" (e.g. "Sug_Mutzar=קופת גמל").
Out of scope → inScope = false, inScopeReason = "<column>=<value> not in scope" for the first rule that failed (check Yatzran, then Subject, then Sug_Mutzar). Still emit the record with excel filled in, match = { how: "none", keyUsed: "none", outcome: "not_found", candidates: 0 } and novidea = { steps: [], maavira: null, mekabelet: null }. Do not call @Novidea for out-of-scope rows.

## 3. Output schema
{
  "meta": {
    "rowKey": string,
    "inScope": boolean,
    "runDate": "YYYY-MM-DD",          // today's date in Asia/Jerusalem
    "subject": string|null,           // the row's Subject
    "rowChange": "new"|"changed"|"deleted",
    "inScopeReason": string
  },
  "excel": {
    "today": ExcelRow|null,           // null when rowChange = "deleted"
    "yesterday": ExcelRow|null        // null when rowChange = "new"
  },
  "match": {
    "how": "ssn+product+policy"|"ssn+transferPolicy"|"none",
    "keyUsed": string,
    "outcome": "matched"|"not_found"|"ambiguous",
    "candidates": number
  },
  "novidea": {
    "steps": Step[],                  // [] when there is no match
    "maavira": Policy|null,           // transferring offer; null when there is no transfer
    "mekabelet": Policy|null          // receiving offer; null when there is no match
  }
}

ExcelRow contains exactly these 30 keys. Copy the raw cell text as a STRING: do not convert dates or numbers, and keep "9/30/26", "0.6", "2_9619" exactly as they appear. Empty cells become null.
Maamad, Status, Maslul1, Maslul2, Maslul3, Maslul4, Maslul5, Subject, Yatzran, Guf_From, Cust_Name, Mis_Zehut, Pull_Type, DepositFee, Sug_Mutzar, Employer_HP, Kabala_Date, Reject_Desc, FirstHafkada, maslul_words, DateExpection, Empolyer_Name, Mispar_Sochen, Kupa_Mekabelet, AccumulationFee, NumMaslulHashkaa, Daily_Report_date, Mispar_Polisa_From, Reject_Detaiks_Text, SchumNiyud_Meshicha_Hafkada_Gvia
(Keep the source spellings "Empolyer_Name", "DateExpection" and "Reject_Detaiks_Text" as they are.)

Policy: one AgentWise_IL__Application__c record (an offer), mapped like this:
{
  "id": Id,
  "ssn": ssn_Insured_Person__c,
  "fundType": Formally__fundtype_description__c,          // e.g. "גמל"
  "carrierId": AgentWise_IL__Selected_Carrier__c,         // Account Id, starts with "001"
  "carrierName": name_carrier__c,                         // exactly as stored, including double spaces
  "depositFee": Formally__Deposit_Fee__c as a STRING,     // e.g. "0"
  "statusNapa": Formally_Status_napa__c,                  // e.g. "בתהליך"
  "agentNumber": NewFundAgentNumber__c,                   // e.g. "2-9619"
  "createdDate": CreatedDate, exactly as returned,        // e.g. "2026-10-05T10:39:26.000+0000"
  "totalTzvira": Formally__Total_tzvira__c as a NUMBER,   // null → 0
  "policyNumber": Formally__policy_number__c,
  "niyudKupaFrom": boolean,       // true when another offer points at this one through Formally__Transferred_to__c, i.e. a transferring offer exists
  "accumulationFees": Formally__Accumulation_fees__c as a NUMBER,   // e.g. 0.6
  "isGemelLaHashkaa": boolean     // true when name_carrier__c contains "גמל להשקעה"
}

Step: one NBPM__Step__c record under the receiving offer's procedure:
{
  "id": Id,
  "name": Name,                                   // e.g. "בקרה מול יצרן", "בדיקה פרמיה ראשונה פנסיוני"
  "status": "Open" | "Closed",                    // NBPM__Status__c: "סגור" → "Closed"; any other value → "Open"
  "dueDate": NBPM__Due_Date__c as "YYYY-MM-DD",
  "remarkOp": Remark_op__c,
  "procedureId": the step's NBPM__Procedure__c Id,
  "originalDueDate": the step's original due-date field as "YYYY-MM-DD",
  "businessDaysFromCreation": null,
  "businessDaysFromLastStatus": null
}
Return every step under the procedure, both open and closed. Sort by originalDueDate ascending.

## 4. Case matching in @Novidea
Only open offers count: AgentWise_IL__Application__c with Formally_Status_napa__c = "בתהליך".
Map Sug_Mutzar to the offer's Formally__fundtype_description__c: "קופת גמל" → "גמל", "קרן השתלמות" → "השתלמות".

A) No transfer (Subject = "הצטרפויות ללא ניוד", or Mispar_Polisa_From is empty):
   how = "ssn+product+policy", keyUsed = "ssn + Sug_Mutzar (+ policy number)".
   Find offers where ssn_Insured_Person__c = Mis_Zehut AND Formally__fundtype_description__c = the mapped product AND Formally__policy_number__c = Kupa_Mekabelet.
   Each found offer is a receiving-offer candidate. maavira = null.

B) With transfer (Subject = "הצטרפויות עם ניוד" and Mispar_Polisa_From is filled):
   how = "ssn+transferPolicy", keyUsed = "ssn + Mispar_Polisa_From (transferring policy)".
   Find transferring offers where ssn_Insured_Person__c = Mis_Zehut AND Formally__policy_number__c = Mispar_Polisa_From.
   For each one, the receiving offer is the record its Formally__Transferred_to__c points to. maavira = the transferring offer, mapped to Policy.

Result:
- candidates = the number of receiving offers found.
- Exactly 1 → outcome = "matched".
- More than 1 → outcome = "ambiguous". Use the one with the latest CreatedDate.
- 0 → outcome = "not_found", mekabelet = null, maavira = null, steps = [].
- mekabelet = the chosen receiving offer, mapped to Policy. steps = all NBPM__Step__c records under its procedure.

## 5. Rules
- Normalize only for comparison: trim, collapse repeated spaces, and treat "_" and "-" as equal. Always output values exactly as the source returned them.
- If an @Novidea call fails, still emit the record with match.outcome = "not_found", novidea empty as above, and meta.inScopeReason = "error: <message>".
- Return an array of transfer records, one per new, changed or deleted row.

## 6. Expected output example (a new no-transfer row, matched)
{"meta":{"rowKey":"23538937|קופת גמל|מיטב דש","inScope":true,"runDate":"2026-10-11","subject":"הצטרפויות ללא ניוד","rowChange":"new","inScopeReason":"Sug_Mutzar=קופת גמל"},"excel":{"today":{"Maamad":"שכיר","Status":"בטיפול","Maslul1":null,"Maslul2":null,"Maslul3":null,"Maslul4":null,"Maslul5":null,"Subject":"הצטרפויות ללא ניוד","Yatzran":"מיטב דש","Guf_From":null,"Cust_Name":"יעקב נחמן","Mis_Zehut":"23538937","Pull_Type":null,"DepositFee":null,"Sug_Mutzar":"קופת גמל","Employer_HP":null,"Kabala_Date":"9/30/26","Reject_Desc":null,"FirstHafkada":"10/10/2026","maslul_words":"מסלול ברירת מחדל","DateExpection":null,"Empolyer_Name":null,"Mispar_Sochen":"2_9619","Kupa_Mekabelet":"032-933-151146-0","AccumulationFee":"0.6","NumMaslulHashkaa":"59999","Daily_Report_date":"10/5/26","Mispar_Polisa_From":null,"Reject_Detaiks_Text":null,"SchumNiyud_Meshicha_Hafkada_Gvia":null},"yesterday":null},"match":{"how":"ssn+product+policy","keyUsed":"ssn + Sug_Mutzar (+ policy number)","outcome":"matched","candidates":1},"novidea":{"steps":[{"id":"a52S900000RBSr3IAH","name":"בקרה מול יצרן","status":"Closed","dueDate":"2026-10-11","remarkOp":null,"procedureId":"a4xS9000002Jf1VIAS","originalDueDate":"2026-10-14","businessDaysFromCreation":null,"businessDaysFromLastStatus":null},{"id":"a52S900000RQ57lIAD","name":"בדיקה פרמיה ראשונה פנסיוני","status":"Open","dueDate":"2026-11-08","remarkOp":null,"procedureId":"a4xS9000002Jf1VIAS","originalDueDate":"2026-11-08","businessDaysFromCreation":null,"businessDaysFromLastStatus":null}],"maavira":null,"mekabelet":{"id":"a09S900000c7C6bIAE","ssn":"23538937","fundType":"גמל","carrierId":"001D00000156nyjIAA","depositFee":"0","statusNapa":"בתהליך","agentNumber":"2-9619","carrierName":"מיטב דש  גמל ופנסיה בע\"מ","createdDate":"2026-10-05T10:39:26.000+0000","totalTzvira":0,"policyNumber":"032-933-151146-0","niyudKupaFrom":false,"accumulationFees":0.6,"isGemelLaHashkaa":false}}}
```
