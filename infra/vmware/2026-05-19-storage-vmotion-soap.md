# Storage vMotion via vCenter SOAP API：不能加 `<priority>` 欄位

> **環境**：vCenter 7.x，線上 VM 不停機搬移 VMDK

---

## 需求

VM 不關機，把 VMDK 從一個 datastore 搬到另一個（Storage vMotion）。

用 vCenter UI 操作沒問題，但需要自動化，所以改用 SOAP API。

---

## 會出錯的寫法

直覺上會這樣寫 `RelocateVM_Task`：

```xml
<urn:RelocateVM_Task>
  <urn:_this type="VirtualMachine">vm-25</urn:_this>
  <urn:spec>
    <urn:datastore type="Datastore">datastore-42</urn:datastore>
    <urn:diskMoveType>moveAllDiskBackingsAndConsolidate</urn:diskMoveType>
    <urn:priority>defaultPriority</urn:priority>
  </urn:spec>
</urn:RelocateVM_Task>
```

結果：vCenter 7.x 直接回錯誤，task 不會建立。

---

## 原因

`<priority>` 欄位在 vCenter 7.x 的 SOAP schema 裡不接受放在 `<spec>` 裡面。

官方文件和網路上大部分的範例是舊版的，有些版本這個欄位放的位置不同，有些版本根本不支援這個欄位放在這個 context。

---

## 正確寫法

拿掉 `<priority>`，其他不變：

```xml
<urn:RelocateVM_Task>
  <urn:_this type="VirtualMachine">vm-25</urn:_this>
  <urn:spec>
    <urn:datastore type="Datastore">datastore-42</urn:datastore>
    <urn:diskMoveType>moveAllDiskBackingsAndConsolidate</urn:diskMoveType>
  </urn:spec>
</urn:RelocateVM_Task>
```

---

## 完整流程備查

**Step 1：取得 session token**

```bash
SESSION=$(curl -sk -u 'administrator@your.domain:Password' \
  -X POST "https://vcenter-ip/rest/com/vmware/cis/session" \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["value"])')
```

**Step 2：取得 VM 的 MoRef**

```bash
curl -sk -H "vmware-api-session-id: $SESSION" \
  "https://vcenter-ip/rest/vcenter/vm" | python3 -m json.tool
# 找到 vm.vm 欄位，格式是 "vm-XX"
```

**Step 3：取得目標 Datastore 的 MoRef**

```bash
curl -sk -H "vmware-api-session-id: $SESSION" \
  "https://vcenter-ip/rest/vcenter/datastore" | python3 -m json.tool
# 找到 datastore.datastore 欄位，格式是 "datastore-XX"
```

**Step 4：送 SOAP 搬移請求**

```bash
curl -sk -u 'administrator@your.domain:Password' \
  -H "Content-Type: text/xml" \
  -H "SOAPAction: urn:vim25/7.0" \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Envelope xmlns="http://schemas.xmlsoap.org/soap/envelope/"
          xmlns:urn="urn:vim25">
  <Body>
    <urn:RelocateVM_Task>
      <urn:_this type="VirtualMachine">vm-25</urn:_this>
      <urn:spec>
        <urn:datastore type="Datastore">datastore-42</urn:datastore>
        <urn:diskMoveType>moveAllDiskBackingsAndConsolidate</urn:diskMoveType>
      </urn:spec>
    </urn:RelocateVM_Task>
  </Body>
</Envelope>' \
  "https://vcenter-ip/sdk"
```

回傳會有一個 task MoRef（`task-XXX`），可以拿去查進度。

**Step 5：查 Task 進度**

```bash
curl -sk -H "vmware-api-session-id: $SESSION" \
  "https://vcenter-ip/rest/vcenter/task/task-XXX" | python3 -m json.tool
# state: RUNNING / SUCCEEDED / FAILED
```

---

## diskMoveType 選項說明

| 值 | 說明 |
|----|------|
| `moveAllDiskBackingsAndConsolidate` | 搬移所有 VMDK 並合併 snapshot（最常用） |
| `moveAllDiskBackingsAndDisallowSharing` | 搬移所有 VMDK，不允許共用 |
| `moveChildMostDiskBacking` | 只搬最新的 snapshot backing（少用） |

一般 Storage vMotion 用 `moveAllDiskBackingsAndConsolidate`。

---

*記錄於 2026-05-19，線上搬移 ES01 VM VMDK 時遇到的問題*
