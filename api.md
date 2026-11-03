# API specification


## API

- [POST /v1/tickets/search](#post-v1ticketssearch)
- [POST /v1/reservations/get](#post-v1reservationsget)
- [POST /v1/reservations/release](#post-v1reservationsrelease)
- [POST /v1/reservations/confirm](#post-v1reservationsconfirm)

## POST /v1/tickets/search

ค้นหาสลากที่ตรง pattern และจองให้ผู้ใช้ทันทีภายใน transaction เดียว

### เงื่อนไขของเส้นนี้

- ต้องระบุ pattern ยาว 6 ตัว เป็นเลข ASCII หรือ * โดย * แทนเลขหนึ่งหลัก
- ต้องระบุ limit 1–100 หากขอ 20 ใบแล้วเลือกได้ 7 ใบ จะจองให้ 7 ใบ ไม่มี total, pagination หรือลำดับผลที่รับประกัน
- key ใช้ร่วมกับ owner จาก JWT ถ้าเคย commit key นี้แล้วและ pattern/limit เดิม ให้คืน reservation ID/ชุดสลากเดิมพร้อมสถานะปัจจุบัน
- key เดิมแต่ payload เปลี่ยนให้ตอบ 409 IDEMPOTENCY_CONFLICT
- การ retry ไม่ต่อเวลา hold และไม่เปิด confirmed/released/expired กลับมา active
- ถ้า key ยังไม่เคย commit และเลือกไม่ได้สักใบ ให้ rollback และไม่บันทึก key/reservation ใหม่ จึงอาจใช้ key เดิมแล้วสำเร็จในครั้งถัดไป
- 409 unavailable อาจเกิดจาก matching rows ถูกล็อก ไม่ได้ยืนยันว่าสลากขายหมด

### Request

#### Headers

| Header | Mandatory | Nullable | ค่า / รายละเอียด |
| --- | --- | --- | --- |
| Authorization | Yes | No | `Bearer <JWT>` ตรวจ signature, issuer และ expiry แล้วใช้ sub เป็นเจ้าของ |
| Content-Type | Yes | No | `application/json` |

#### Body fields

รับ JSON object เดียว ขนาดไม่เกิน 4 KiB ไม่รับข้อมูลธุรกิจจาก query/path และไม่รับ field ที่ไม่รู้จักหรือ field ซ้ำ

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `pattern` | string | Yes | No | คำค้นยาว 6 ตัว ใช้เลข ASCII 0–9 หรือ * เท่านั้น แต่ละ * แทนเลขหนึ่งหลัก ไม่ตัดช่องว่างให้ |
| `limit` | integer | Yes | No | จำนวนใบสูงสุดที่ต้องการ ต้องระบุ 1–100 ไม่มีค่าเริ่มต้น ระบบอาจคืนให้น้อยกว่าที่ขอได้ |
| `idempotency_key` | string | Yes | No | รหัสคำขอที่ client สร้าง ใช้ค่าเดิมเมื่อ retry สำหรับผู้ใช้คนเดิม ห้ามว่าง/มีแต่ช่องว่าง/มี NUL และยาวไม่เกิน 128 UTF-8 bytes ไม่แก้ตัวใหญ่เล็กหรือตัดช่องว่างก่อนเทียบ |

#### ตัวอย่าง request JSON

```json
{
  "pattern": "****23",
  "limit": 2,
  "idempotency_key": "search-user-a-001"
}
```

### Response เมื่อสำเร็จ

#### Response fields

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `data` | object | Yes | No | ผลการค้นหาและจอง ประกอบด้วย reservation กับ replayed |
| `data.reservation` | object | Yes | No | การจองเดิมพร้อมสถานะปัจจุบัน หากถึงเวลาหมดอายุให้จัดการ expiry ก่อนคืนผล |
| `data.reservation.id` | string (UUID) | Yes | No | UUID ของการจอง สร้างใหม่ต่อการจัดสรรครั้งใหม่ ไม่เอา ID เก่ากลับมาใช้ |
| `data.reservation.user_id` | string | Yes | No | เจ้าของการจองจาก sub ใน JWT ไม่รับให้ client เลือกผ่าน body ไม่เกิน 256 UTF-8 bytes |
| `data.reservation.pattern` | string | Yes | No | pattern ที่ใช้สร้างการจอง ยาว 6 ตัวเลข/* และไม่เปลี่ยนจากเดิม |
| `data.reservation.requested_count` | integer | Yes | No | limit ที่ผู้ใช้ขอไว้ครั้งแรก อยู่ระหว่าง 1–100 และไม่เปลี่ยนเมื่อ retry |
| `data.reservation.allocated_count` | integer | Yes | No | จำนวนใบที่จองได้จริง อยู่ระหว่าง 1–100 ต้องเท่ากับ tickets.length และไม่เกิน requested_count |
| `data.reservation.status` | string | Yes | No | active=กำลัง hold, confirmed=ยืนยันแล้ว, released=ผู้ใช้ยกเลิก, expired=หมดเวลา |
| `data.reservation.created_at` | string (UTC RFC 3339) | Yes | No | เวลาสร้างจากฐานข้อมูล ส่งเป็น UTC รูปแบบ RFC 3339 เช่น 2026-11-02T10:00:00Z |
| `data.reservation.expires_at` | string (UTC RFC 3339) | Yes | No | เวลาสิ้นสุด hold ต้องหลัง created_at หาก confirmed แล้วเวลานี้ไม่ทำให้การยืนยันหมดอายุ |
| `data.reservation.updated_at` | string (UTC RFC 3339) | Yes | No | เวลาที่สถานะเปลี่ยนล่าสุดจากฐานข้อมูล ส่งเป็น UTC RFC 3339 การ retry ที่ไม่เปลี่ยนสถานะไม่แก้เวลานี้ |
| `data.reservation.tickets` | object[] | Yes | No | สลากที่เคยได้จากการจองนี้ 1–100 ใบ ไม่รับประกันลำดับ เมื่อ released/expired แล้วยังคงคืนรายการเดิมเป็นประวัติ |
| `data.reservation.tickets[].ticket_id` | string | Yes เมื่อมีรายการนี้ | No | ID ของสลากใบจริง DB เก็บเป็น bigint แต่ API ส่งเป็น string ของเลขจำนวนเต็มบวก เพื่อไม่ให้ client อ่านเลขคลาดเคลื่อน |
| `data.reservation.tickets[].number` | string | Yes เมื่อมีรายการนี้ | No | เลขสลาก 6 หลัก เก็บเป็น string เพื่อรักษาศูนย์นำหน้า คนละ ticket_id อาจมี number เดียวกันได้ |
| `data.replayed` | boolean | Yes | No | false เมื่อสร้างใหม่และตอบ 201; true เมื่อได้รายการเดิมจาก key เดิมและตอบ 200 ไม่ต่อเวลา hold |
| `request_id` | string (UUID) | Yes | No | UUID ที่ server สร้างสำหรับติดตาม HTTP request ครั้งนี้ จึงเปลี่ยนได้แม้ retry การจองเดิม |

#### 201 Created

สร้างและ commit การจองใหม่แล้ว `replayed=false`

```json
{
  "data": {
    "reservation": {
      "id": "ecad59f1-a023-40c1-9213-ef7f7cf396a0",
      "user_id": "user-a",
      "pattern": "****23",
      "requested_count": 2,
      "allocated_count": 2,
      "status": "active",
      "created_at": "2026-11-02T10:00:00Z",
      "expires_at": "2026-11-02T10:05:00Z",
      "updated_at": "2026-11-02T10:00:00Z",
      "tickets": [
        {
          "ticket_id": "24",
          "number": "000023"
        },
        {
          "ticket_id": "124",
          "number": "000123"
        }
      ]
    },
    "replayed": false
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

#### 200 OK — retry คำขอเดิม

ใช้ request body เดิม เมื่อ key เคย commit แล้วจะได้ `replayed=true` ไม่จองเพิ่ม ตัวอย่างยัง active แต่ของจริงอาจคืน terminal state ปัจจุบันได้

```json
{
  "data": {
    "reservation": {
      "id": "ecad59f1-a023-40c1-9213-ef7f7cf396a0",
      "user_id": "user-a",
      "pattern": "****23",
      "requested_count": 2,
      "allocated_count": 2,
      "status": "active",
      "created_at": "2026-11-02T10:00:00Z",
      "expires_at": "2026-11-02T10:05:00Z",
      "updated_at": "2026-11-02T10:00:00Z",
      "tickets": [
        {
          "ticket_id": "24",
          "number": "000023"
        },
        {
          "ticket_id": "124",
          "number": "000123"
        }
      ]
    },
    "replayed": true
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

### Error ของเส้นนี้

| HTTP | error.code | เกิดเมื่อ | วิธีจัดการ |
| --- | --- | --- | --- |
| 400 | `INVALID_REQUEST` | body, content type, field หรือรูปแบบข้อมูลไม่ถูกต้อง | แก้ข้อมูลก่อนส่งใหม่ |
| 401 | `UNAUTHENTICATED` | ไม่มี JWT ที่ตรวจผ่าน | ใช้ token ที่ถูกต้อง |
| 409 | `ALLOCATION_UNAVAILABLE` | ขณะนี้เลือกสลากที่ตรง pattern ไม่ได้ อาจเป็นเพราะแถวติด lock | รอแล้วลองใหม่แบบจำกัดจำนวน ใช้ key เดิมได้ ไม่สรุปว่าขายหมด |
| 409 | `IDEMPOTENCY_CONFLICT` | owner/key นี้เคยใช้แล้ว แต่ pattern หรือ limit ต่างจากเดิม | ใช้ payload เดิมกับ key เดิม หรือ key ใหม่เมื่อเป็นความตั้งใจจองครั้งใหม่ |
| 429 | `RATE_LIMITED` | ส่งคำขอเกินเกณฑ์ที่กำหนด | รอตาม Retry-After และจำกัดจำนวนครั้งที่ลองใหม่ |
| 500 | `INTERNAL_ERROR` | ข้อผิดพลาดภายใน โดยไม่เปิดเผย SQL หรือข้อมูลลับ | ใช้ request_id ให้ทีมตรวจปัญหา ไม่เริ่มการจองใหม่โดยเดาว่างานเดิมล้มเหลว |
| 503 | `TEMPORARILY_UNAVAILABLE` | DB, pool หรือ lock ยังไม่พร้อมรับงานนี้ | รอตาม Retry-After แล้วส่งคำขอเดิมแบบจำกัดจำนวน |
| 504 | `DEADLINE_EXCEEDED` | คำขอใช้เวลาเกินกำหนด อาจ commit แล้วแต่ตอบไม่ทัน | retry search ด้วย owner/key/payload เดิม ห้ามสร้าง key ใหม่อัตโนมัติ เพราะการจองเดิมอาจสำเร็จแล้ว |

429/503 ต้องมี Retry-After เป็นจำนวนวินาทีขั้นต่ำที่แนะนำให้รอ จำกัดจำนวน retries และสุ่มเวลารอเพิ่มเล็กน้อย เพื่อไม่ให้คำขอกลับมาพร้อมกัน หาก network ขาดก่อนมี HTTP response ให้ถือว่ายังไม่รู้ผล ไม่ใช่ยืนยันว่างานล้มเหลว

#### Error response fields

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `error` | object | Yes | No | รหัสและข้อความอธิบายข้อผิดพลาด รวมรายละเอียด field ถ้ามี |
| `error.code` | string | Yes | No | รหัสคงที่สำหรับให้ client เลือกวิธีจัดการร่วมกับ HTTP status ดูรายการในตาราง error codes |
| `error.message` | string | Yes | No | คำอธิบายที่ไม่เปิดเผย SQL, secret หรือ token |
| `error.details` | object[] | No | No | รายละเอียด field ที่ไม่ผ่าน validation เป็น optional field ละได้เมื่อไม่เกี่ยวกับ field และห้ามเป็น null |
| `error.details[].field` | string | Yes เมื่อมีรายการนี้ | No | ชื่อหรือ path ของ field ที่ไม่ถูกต้อง เช่น pattern |
| `error.details[].message` | string | Yes เมื่อมีรายการนี้ | No | คำอธิบายที่อ่านให้ผู้ใช้หรือ developer เข้าใจ ไม่ควรให้ client แยกข้อความนี้เพื่อเลือก logic |
| `request_id` | string (UUID) | Yes | No | UUID สำหรับติดตาม request ที่ไม่สำเร็จด้วย |

#### ตัวอย่าง error — 400 Bad Request — validation

```json
{
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Request validation failed.",
    "details": [
      {
        "field": "pattern",
        "message": "Must contain exactly six ASCII digits or *."
      }
    ]
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

#### ตัวอย่าง error — 409 Conflict — ยังเลือกสลากไม่ได้

```json
{
  "error": {
    "code": "ALLOCATION_UNAVAILABLE",
    "message": "No matching tickets can be allocated at this time."
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

### ข้อควรระวังตอน implement เส้นนี้

ตรวจ UTF-8 byte length ของ key เพิ่มจากจำนวนตัวอักษร และให้ allocated_count เท่ากับ tickets.length ไม่เกิน requested_count ห้ามคืน ticket IDs ก่อน commit

## POST /v1/reservations/get

ดูการจองของผู้ใช้ที่อยู่ใน JWT รวมถึงประวัติของการจองที่สิ้นสุดแล้ว

### เงื่อนไขของเส้นนี้

- ต้องส่ง reservation_id ใน body และดูได้เฉพาะของตัวเอง
- UUID ไม่ถูกต้องตอบ 400 ไม่พบหรือเป็นของคนอื่นตอบ 404 แบบเดียวกัน
- ถ้า active เลย deadline ให้คืนสลากและบันทึก expired ใน transaction ก่อนตอบ 200
- tickets ของ released/expired เป็นประวัติ ไม่ใช่สิทธิ์จองที่ยังใช้งานได้

### Request

#### Headers

| Header | Mandatory | Nullable | ค่า / รายละเอียด |
| --- | --- | --- | --- |
| Authorization | Yes | No | `Bearer <JWT>` ตรวจ signature, issuer และ expiry แล้วใช้ sub เป็นเจ้าของ |
| Content-Type | Yes | No | `application/json` |

#### Body fields

รับ JSON object เดียว ขนาดไม่เกิน 4 KiB ไม่รับข้อมูลธุรกิจจาก query/path และไม่รับ field ที่ไม่รู้จักหรือ field ซ้ำ

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `reservation_id` | string (UUID) | Yes | No | UUID ของการจองที่ต้องการใช้งาน ต้องเป็นของผู้ใช้ที่อยู่ใน JWT |

#### ตัวอย่าง request JSON

```json
{
  "reservation_id": "ecad59f1-a023-40c1-9213-ef7f7cf396a0"
}
```

### Response เมื่อสำเร็จ

#### Response fields

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `data` | object | Yes | No | ข้อมูลการจองของเจ้าของ get อาจคืน status=expired ด้วย HTTP 200 ได้ |
| `data.id` | string (UUID) | Yes | No | UUID ของการจอง สร้างใหม่ต่อการจัดสรรครั้งใหม่ ไม่เอา ID เก่ากลับมาใช้ |
| `data.user_id` | string | Yes | No | เจ้าของการจองจาก sub ใน JWT ไม่รับให้ client เลือกผ่าน body ไม่เกิน 256 UTF-8 bytes |
| `data.pattern` | string | Yes | No | pattern ที่ใช้สร้างการจอง ยาว 6 ตัวเลข/* และไม่เปลี่ยนจากเดิม |
| `data.requested_count` | integer | Yes | No | limit ที่ผู้ใช้ขอไว้ครั้งแรก อยู่ระหว่าง 1–100 และไม่เปลี่ยนเมื่อ retry |
| `data.allocated_count` | integer | Yes | No | จำนวนใบที่จองได้จริง อยู่ระหว่าง 1–100 ต้องเท่ากับ tickets.length และไม่เกิน requested_count |
| `data.status` | string | Yes | No | active=กำลัง hold, confirmed=ยืนยันแล้ว, released=ผู้ใช้ยกเลิก, expired=หมดเวลา |
| `data.created_at` | string (UTC RFC 3339) | Yes | No | เวลาสร้างจากฐานข้อมูล ส่งเป็น UTC รูปแบบ RFC 3339 เช่น 2026-11-02T10:00:00Z |
| `data.expires_at` | string (UTC RFC 3339) | Yes | No | เวลาสิ้นสุด hold ต้องหลัง created_at หาก confirmed แล้วเวลานี้ไม่ทำให้การยืนยันหมดอายุ |
| `data.updated_at` | string (UTC RFC 3339) | Yes | No | เวลาที่สถานะเปลี่ยนล่าสุดจากฐานข้อมูล ส่งเป็น UTC RFC 3339 การ retry ที่ไม่เปลี่ยนสถานะไม่แก้เวลานี้ |
| `data.tickets` | object[] | Yes | No | สลากที่เคยได้จากการจองนี้ 1–100 ใบ ไม่รับประกันลำดับ เมื่อ released/expired แล้วยังคงคืนรายการเดิมเป็นประวัติ |
| `data.tickets[].ticket_id` | string | Yes เมื่อมีรายการนี้ | No | ID ของสลากใบจริง DB เก็บเป็น bigint แต่ API ส่งเป็น string ของเลขจำนวนเต็มบวก เพื่อไม่ให้ client อ่านเลขคลาดเคลื่อน |
| `data.tickets[].number` | string | Yes เมื่อมีรายการนี้ | No | เลขสลาก 6 หลัก เก็บเป็น string เพื่อรักษาศูนย์นำหน้า คนละ ticket_id อาจมี number เดียวกันได้ |
| `request_id` | string (UUID) | Yes | No | UUID ที่ server สร้างสำหรับติดตาม HTTP request ครั้งนี้ |

#### 200 OK — active

คืนข้อมูลการจองปัจจุบัน

```json
{
  "data": {
    "id": "ecad59f1-a023-40c1-9213-ef7f7cf396a0",
    "user_id": "user-a",
    "pattern": "****23",
    "requested_count": 2,
    "allocated_count": 2,
    "status": "active",
    "created_at": "2026-11-02T10:00:00Z",
    "expires_at": "2026-11-02T10:05:00Z",
    "updated_at": "2026-11-02T10:00:00Z",
    "tickets": [
      {
        "ticket_id": "24",
        "number": "000023"
      },
      {
        "ticket_id": "124",
        "number": "000123"
      }
    ]
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

#### 200 OK — expired

แม้หมดเวลาแล้วเจ้าของยังอ่านประวัติได้ จึงตอบ 200 ไม่ใช่ 410 สำหรับเส้น get

```json
{
  "data": {
    "id": "ecad59f1-a023-40c1-9213-ef7f7cf396a0",
    "user_id": "user-a",
    "pattern": "****23",
    "requested_count": 2,
    "allocated_count": 2,
    "status": "expired",
    "created_at": "2026-11-02T10:00:00Z",
    "expires_at": "2026-11-02T10:05:00Z",
    "updated_at": "2026-11-02T10:05:00Z",
    "tickets": [
      {
        "ticket_id": "24",
        "number": "000023"
      },
      {
        "ticket_id": "124",
        "number": "000123"
      }
    ]
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

### Error ของเส้นนี้

| HTTP | error.code | เกิดเมื่อ | วิธีจัดการ |
| --- | --- | --- | --- |
| 400 | `INVALID_REQUEST` | body, content type, field หรือรูปแบบข้อมูลไม่ถูกต้อง | แก้ข้อมูลก่อนส่งใหม่ |
| 401 | `UNAUTHENTICATED` | ไม่มี JWT ที่ตรวจผ่าน | ใช้ token ที่ถูกต้อง |
| 404 | `RESERVATION_NOT_FOUND` | ไม่พบ reservation หรือผู้เรียกไม่ใช่เจ้าของ | ไม่เผยว่าเป็นของผู้ใช้คนอื่น และไม่เปลี่ยนข้อมูล |
| 429 | `RATE_LIMITED` | ส่งคำขอเกินเกณฑ์ที่กำหนด | รอตาม Retry-After และจำกัดจำนวนครั้งที่ลองใหม่ |
| 500 | `INTERNAL_ERROR` | ข้อผิดพลาดภายใน โดยไม่เปิดเผย SQL หรือข้อมูลลับ | ใช้ request_id ให้ทีมตรวจปัญหา ไม่เริ่มการจองใหม่โดยเดาว่างานเดิมล้มเหลว |
| 503 | `TEMPORARILY_UNAVAILABLE` | DB, pool หรือ lock ยังไม่พร้อมรับงานนี้ | รอตาม Retry-After แล้วส่งคำขอเดิมแบบจำกัดจำนวน |
| 504 | `DEADLINE_EXCEEDED` | คำขอใช้เวลาเกินกำหนด อาจ commit แล้วแต่ตอบไม่ทัน | retry get ด้วย reservation_id เดิม ไม่เริ่ม search ใหม่เพื่อทดแทนการอ่านที่ยังไม่รู้ผล |

429/503 ต้องมี Retry-After เป็นจำนวนวินาทีขั้นต่ำที่แนะนำให้รอ จำกัดจำนวน retries และสุ่มเวลารอเพิ่มเล็กน้อย เพื่อไม่ให้คำขอกลับมาพร้อมกัน หาก network ขาดก่อนมี HTTP response ให้ถือว่ายังไม่รู้ผล ไม่ใช่ยืนยันว่างานล้มเหลว

#### Error response fields

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `error` | object | Yes | No | รหัสและข้อความอธิบายข้อผิดพลาด รวมรายละเอียด field ถ้ามี |
| `error.code` | string | Yes | No | รหัสคงที่สำหรับให้ client เลือกวิธีจัดการร่วมกับ HTTP status ดูรายการในตาราง error codes |
| `error.message` | string | Yes | No | คำอธิบายที่ไม่เปิดเผย SQL, secret หรือ token |
| `error.details` | object[] | No | No | รายละเอียด field ที่ไม่ผ่าน validation เป็น optional field ละได้เมื่อไม่เกี่ยวกับ field และห้ามเป็น null |
| `error.details[].field` | string | Yes เมื่อมีรายการนี้ | No | ชื่อหรือ path ของ field ที่ไม่ถูกต้อง เช่น pattern |
| `error.details[].message` | string | Yes เมื่อมีรายการนี้ | No | คำอธิบายที่อ่านให้ผู้ใช้หรือ developer เข้าใจ ไม่ควรให้ client แยกข้อความนี้เพื่อเลือก logic |
| `request_id` | string (UUID) | Yes | No | UUID สำหรับติดตาม request ที่ไม่สำเร็จด้วย |

#### ตัวอย่าง error — 404 Not Found

```json
{
  "error": {
    "code": "RESERVATION_NOT_FOUND",
    "message": "Reservation not found."
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

### ข้อควรระวังตอน implement เส้นนี้

กรอง owner ใน query และล็อก reservation ก่อนตรวจเวลา หากมี lazy expiry ต้อง commit การคืนสลากก่อนตอบ response

## POST /v1/reservations/release

ยกเลิก active hold และคืนสลากให้ผู้ใช้อื่นค้นหาและจองได้

### เงื่อนไขของเส้นนี้

- ต้องเป็นเจ้าของและ reservation ยัง active ก่อน deadline
- เปลี่ยน reservation เป็น released พร้อมคืน reserved tickets เป็น available และล้าง current reference ใน transaction เดียว
- ถ้า released อยู่แล้วตอบ 200 แบบเดิม โดยไม่คืนสลากซ้ำให้มีผลเพิ่ม
- ถ้า confirmed แล้วตอบ 409 STATE_CONFLICT หาก expired หรือเลย deadline ให้บันทึก expiry แล้วตอบ 410
- เก็บ history ไว้ และห้ามใช้ reservation เก่าไปคืนสลากที่ถูกจองใหม่แล้ว

### Request

#### Headers

| Header | Mandatory | Nullable | ค่า / รายละเอียด |
| --- | --- | --- | --- |
| Authorization | Yes | No | `Bearer <JWT>` ตรวจ signature, issuer และ expiry แล้วใช้ sub เป็นเจ้าของ |
| Content-Type | Yes | No | `application/json` |

#### Body fields

รับ JSON object เดียว ขนาดไม่เกิน 4 KiB ไม่รับข้อมูลธุรกิจจาก query/path และไม่รับ field ที่ไม่รู้จักหรือ field ซ้ำ

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `reservation_id` | string (UUID) | Yes | No | UUID ของการจองที่ต้องการใช้งาน ต้องเป็นของผู้ใช้ที่อยู่ใน JWT |

#### ตัวอย่าง request JSON

```json
{
  "reservation_id": "ecad59f1-a023-40c1-9213-ef7f7cf396a0"
}
```

### Response เมื่อสำเร็จ

#### Response fields

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `data` | object | Yes | No | ข้อมูลการจองของเจ้าของ get อาจคืน status=expired ด้วย HTTP 200 ได้ |
| `data.id` | string (UUID) | Yes | No | UUID ของการจอง สร้างใหม่ต่อการจัดสรรครั้งใหม่ ไม่เอา ID เก่ากลับมาใช้ |
| `data.user_id` | string | Yes | No | เจ้าของการจองจาก sub ใน JWT ไม่รับให้ client เลือกผ่าน body ไม่เกิน 256 UTF-8 bytes |
| `data.pattern` | string | Yes | No | pattern ที่ใช้สร้างการจอง ยาว 6 ตัวเลข/* และไม่เปลี่ยนจากเดิม |
| `data.requested_count` | integer | Yes | No | limit ที่ผู้ใช้ขอไว้ครั้งแรก อยู่ระหว่าง 1–100 และไม่เปลี่ยนเมื่อ retry |
| `data.allocated_count` | integer | Yes | No | จำนวนใบที่จองได้จริง อยู่ระหว่าง 1–100 ต้องเท่ากับ tickets.length และไม่เกิน requested_count |
| `data.status` | string | Yes | No | active=กำลัง hold, confirmed=ยืนยันแล้ว, released=ผู้ใช้ยกเลิก, expired=หมดเวลา |
| `data.created_at` | string (UTC RFC 3339) | Yes | No | เวลาสร้างจากฐานข้อมูล ส่งเป็น UTC รูปแบบ RFC 3339 เช่น 2026-11-02T10:00:00Z |
| `data.expires_at` | string (UTC RFC 3339) | Yes | No | เวลาสิ้นสุด hold ต้องหลัง created_at หาก confirmed แล้วเวลานี้ไม่ทำให้การยืนยันหมดอายุ |
| `data.updated_at` | string (UTC RFC 3339) | Yes | No | เวลาที่สถานะเปลี่ยนล่าสุดจากฐานข้อมูล ส่งเป็น UTC RFC 3339 การ retry ที่ไม่เปลี่ยนสถานะไม่แก้เวลานี้ |
| `data.tickets` | object[] | Yes | No | สลากที่เคยได้จากการจองนี้ 1–100 ใบ ไม่รับประกันลำดับ เมื่อ released/expired แล้วยังคงคืนรายการเดิมเป็นประวัติ |
| `data.tickets[].ticket_id` | string | Yes เมื่อมีรายการนี้ | No | ID ของสลากใบจริง DB เก็บเป็น bigint แต่ API ส่งเป็น string ของเลขจำนวนเต็มบวก เพื่อไม่ให้ client อ่านเลขคลาดเคลื่อน |
| `data.tickets[].number` | string | Yes เมื่อมีรายการนี้ | No | เลขสลาก 6 หลัก เก็บเป็น string เพื่อรักษาศูนย์นำหน้า คนละ ticket_id อาจมี number เดียวกันได้ |
| `request_id` | string (UUID) | Yes | No | UUID ที่ server สร้างสำหรับติดตาม HTTP request ครั้งนี้ |

#### 200 OK — released

ยกเลิกสำเร็จ หรือรายการนี้ released อยู่แล้ว tickets ใน response ยังคงเป็นประวัติของชุดเดิม

```json
{
  "data": {
    "id": "ecad59f1-a023-40c1-9213-ef7f7cf396a0",
    "user_id": "user-a",
    "pattern": "****23",
    "requested_count": 2,
    "allocated_count": 2,
    "status": "released",
    "created_at": "2026-11-02T10:00:00Z",
    "expires_at": "2026-11-02T10:05:00Z",
    "updated_at": "2026-11-02T10:02:00Z",
    "tickets": [
      {
        "ticket_id": "24",
        "number": "000023"
      },
      {
        "ticket_id": "124",
        "number": "000123"
      }
    ]
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

### Error ของเส้นนี้

| HTTP | error.code | เกิดเมื่อ | วิธีจัดการ |
| --- | --- | --- | --- |
| 400 | `INVALID_REQUEST` | body, content type, field หรือรูปแบบข้อมูลไม่ถูกต้อง | แก้ข้อมูลก่อนส่งใหม่ |
| 401 | `UNAUTHENTICATED` | ไม่มี JWT ที่ตรวจผ่าน | ใช้ token ที่ถูกต้อง |
| 404 | `RESERVATION_NOT_FOUND` | ไม่พบ reservation หรือผู้เรียกไม่ใช่เจ้าของ | ไม่เผยว่าเป็นของผู้ใช้คนอื่น และไม่เปลี่ยนข้อมูล |
| 409 | `STATE_CONFLICT` | reservation เป็น confirmed แล้ว จึง release ไม่ได้ | ตรวจสถานะปัจจุบัน ไม่ส่ง action ที่ขัดแย้งซ้ำ |
| 410 | `RESERVATION_EXPIRED` | การจองหมดเวลาหรือเคยเป็น expired แล้ว | หากต้องการจองครั้งใหม่ให้เริ่ม search ด้วย key ใหม่ |
| 429 | `RATE_LIMITED` | ส่งคำขอเกินเกณฑ์ที่กำหนด | รอตาม Retry-After และจำกัดจำนวนครั้งที่ลองใหม่ |
| 500 | `INTERNAL_ERROR` | ข้อผิดพลาดภายใน โดยไม่เปิดเผย SQL หรือข้อมูลลับ | ใช้ request_id ให้ทีมตรวจปัญหา ไม่เริ่มการจองใหม่โดยเดาว่างานเดิมล้มเหลว |
| 503 | `TEMPORARILY_UNAVAILABLE` | DB, pool หรือ lock ยังไม่พร้อมรับงานนี้ | รอตาม Retry-After แล้วส่งคำขอเดิมแบบจำกัดจำนวน |
| 504 | `DEADLINE_EXCEEDED` | คำขอใช้เวลาเกินกำหนด อาจ commit แล้วแต่ตอบไม่ทัน | retry release ด้วย reservation_id เดิม เพื่อรับสถานะปัจจุบัน ไม่เริ่ม search ใหม่อัตโนมัติ |

429/503 ต้องมี Retry-After เป็นจำนวนวินาทีขั้นต่ำที่แนะนำให้รอ จำกัดจำนวน retries และสุ่มเวลารอเพิ่มเล็กน้อย เพื่อไม่ให้คำขอกลับมาพร้อมกัน หาก network ขาดก่อนมี HTTP response ให้ถือว่ายังไม่รู้ผล ไม่ใช่ยืนยันว่างานล้มเหลว

#### Error response fields

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `error` | object | Yes | No | รหัสและข้อความอธิบายข้อผิดพลาด รวมรายละเอียด field ถ้ามี |
| `error.code` | string | Yes | No | รหัสคงที่สำหรับให้ client เลือกวิธีจัดการร่วมกับ HTTP status ดูรายการในตาราง error codes |
| `error.message` | string | Yes | No | คำอธิบายที่ไม่เปิดเผย SQL, secret หรือ token |
| `error.details` | object[] | No | No | รายละเอียด field ที่ไม่ผ่าน validation เป็น optional field ละได้เมื่อไม่เกี่ยวกับ field และห้ามเป็น null |
| `error.details[].field` | string | Yes เมื่อมีรายการนี้ | No | ชื่อหรือ path ของ field ที่ไม่ถูกต้อง เช่น pattern |
| `error.details[].message` | string | Yes เมื่อมีรายการนี้ | No | คำอธิบายที่อ่านให้ผู้ใช้หรือ developer เข้าใจ ไม่ควรให้ client แยกข้อความนี้เพื่อเลือก logic |
| `request_id` | string (UUID) | Yes | No | UUID สำหรับติดตาม request ที่ไม่สำเร็จด้วย |

#### ตัวอย่าง error — 409 Conflict — ยืนยันไปแล้ว

```json
{
  "error": {
    "code": "STATE_CONFLICT",
    "message": "Reservation state conflicts with this action."
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

### ข้อควรระวังตอน implement เส้นนี้

ล็อก reservation ก่อน ticket และ update เฉพาะ current_reservation_id ที่ตรงและ status=reserved ถ้าพบว่าหมดเวลาให้ commit expiry ก่อนตอบ 410

## POST /v1/reservations/confirm

ยืนยันการจัดสรรให้เป็น confirmed และเปลี่ยนสลากเป็น allocated เส้นนี้เป็นตัวเลือกเพิ่มเติมจากโจทย์ ไม่ได้แปลว่าชำระเงินแล้ว

### เงื่อนไขของเส้นนี้

- ต้องเป็นเจ้าของและ reservation ยัง active ก่อน deadline
- เปลี่ยน reservation เป็น confirmed และ tickets เป็น allocated ภายใน transaction เดียว
- ถ้า confirmed อยู่แล้วตอบ 200 โดยไม่สร้างการจัดสรรเพิ่ม
- ถ้า released แล้วตอบ 409 STATE_CONFLICT ถ้า expired หรือเลย deadline ให้บันทึก expiry แล้วตอบ 410
- confirmed ไม่หมดอายุอัตโนมัติตาม expires_at และ release กลับไม่ได้

### Request

#### Headers

| Header | Mandatory | Nullable | ค่า / รายละเอียด |
| --- | --- | --- | --- |
| Authorization | Yes | No | `Bearer <JWT>` ตรวจ signature, issuer และ expiry แล้วใช้ sub เป็นเจ้าของ |
| Content-Type | Yes | No | `application/json` |

#### Body fields

รับ JSON object เดียว ขนาดไม่เกิน 4 KiB ไม่รับข้อมูลธุรกิจจาก query/path และไม่รับ field ที่ไม่รู้จักหรือ field ซ้ำ

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `reservation_id` | string (UUID) | Yes | No | UUID ของการจองที่ต้องการใช้งาน ต้องเป็นของผู้ใช้ที่อยู่ใน JWT |

#### ตัวอย่าง request JSON

```json
{
  "reservation_id": "ecad59f1-a023-40c1-9213-ef7f7cf396a0"
}
```

### Response เมื่อสำเร็จ

#### Response fields

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `data` | object | Yes | No | ข้อมูลการจองของเจ้าของ get อาจคืน status=expired ด้วย HTTP 200 ได้ |
| `data.id` | string (UUID) | Yes | No | UUID ของการจอง สร้างใหม่ต่อการจัดสรรครั้งใหม่ ไม่เอา ID เก่ากลับมาใช้ |
| `data.user_id` | string | Yes | No | เจ้าของการจองจาก sub ใน JWT ไม่รับให้ client เลือกผ่าน body ไม่เกิน 256 UTF-8 bytes |
| `data.pattern` | string | Yes | No | pattern ที่ใช้สร้างการจอง ยาว 6 ตัวเลข/* และไม่เปลี่ยนจากเดิม |
| `data.requested_count` | integer | Yes | No | limit ที่ผู้ใช้ขอไว้ครั้งแรก อยู่ระหว่าง 1–100 และไม่เปลี่ยนเมื่อ retry |
| `data.allocated_count` | integer | Yes | No | จำนวนใบที่จองได้จริง อยู่ระหว่าง 1–100 ต้องเท่ากับ tickets.length และไม่เกิน requested_count |
| `data.status` | string | Yes | No | active=กำลัง hold, confirmed=ยืนยันแล้ว, released=ผู้ใช้ยกเลิก, expired=หมดเวลา |
| `data.created_at` | string (UTC RFC 3339) | Yes | No | เวลาสร้างจากฐานข้อมูล ส่งเป็น UTC รูปแบบ RFC 3339 เช่น 2026-11-02T10:00:00Z |
| `data.expires_at` | string (UTC RFC 3339) | Yes | No | เวลาสิ้นสุด hold ต้องหลัง created_at หาก confirmed แล้วเวลานี้ไม่ทำให้การยืนยันหมดอายุ |
| `data.updated_at` | string (UTC RFC 3339) | Yes | No | เวลาที่สถานะเปลี่ยนล่าสุดจากฐานข้อมูล ส่งเป็น UTC RFC 3339 การ retry ที่ไม่เปลี่ยนสถานะไม่แก้เวลานี้ |
| `data.tickets` | object[] | Yes | No | สลากที่เคยได้จากการจองนี้ 1–100 ใบ ไม่รับประกันลำดับ เมื่อ released/expired แล้วยังคงคืนรายการเดิมเป็นประวัติ |
| `data.tickets[].ticket_id` | string | Yes เมื่อมีรายการนี้ | No | ID ของสลากใบจริง DB เก็บเป็น bigint แต่ API ส่งเป็น string ของเลขจำนวนเต็มบวก เพื่อไม่ให้ client อ่านเลขคลาดเคลื่อน |
| `data.tickets[].number` | string | Yes เมื่อมีรายการนี้ | No | เลขสลาก 6 หลัก เก็บเป็น string เพื่อรักษาศูนย์นำหน้า คนละ ticket_id อาจมี number เดียวกันได้ |
| `request_id` | string (UUID) | Yes | No | UUID ที่ server สร้างสำหรับติดตาม HTTP request ครั้งนี้ |

#### 200 OK — confirmed

ยืนยันสำเร็จ หรือรายการนี้ confirmed อยู่แล้ว ไม่เกี่ยวกับการรับชำระเงิน

```json
{
  "data": {
    "id": "ecad59f1-a023-40c1-9213-ef7f7cf396a0",
    "user_id": "user-a",
    "pattern": "****23",
    "requested_count": 2,
    "allocated_count": 2,
    "status": "confirmed",
    "created_at": "2026-11-02T10:00:00Z",
    "expires_at": "2026-11-02T10:05:00Z",
    "updated_at": "2026-11-02T10:02:00Z",
    "tickets": [
      {
        "ticket_id": "24",
        "number": "000023"
      },
      {
        "ticket_id": "124",
        "number": "000123"
      }
    ]
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

### Error ของเส้นนี้

| HTTP | error.code | เกิดเมื่อ | วิธีจัดการ |
| --- | --- | --- | --- |
| 400 | `INVALID_REQUEST` | body, content type, field หรือรูปแบบข้อมูลไม่ถูกต้อง | แก้ข้อมูลก่อนส่งใหม่ |
| 401 | `UNAUTHENTICATED` | ไม่มี JWT ที่ตรวจผ่าน | ใช้ token ที่ถูกต้อง |
| 404 | `RESERVATION_NOT_FOUND` | ไม่พบ reservation หรือผู้เรียกไม่ใช่เจ้าของ | ไม่เผยว่าเป็นของผู้ใช้คนอื่น และไม่เปลี่ยนข้อมูล |
| 409 | `STATE_CONFLICT` | reservation เป็น released แล้ว จึง confirm ไม่ได้ | ตรวจสถานะปัจจุบัน ไม่ส่ง action ที่ขัดแย้งซ้ำ |
| 410 | `RESERVATION_EXPIRED` | การจองหมดเวลาหรือเคยเป็น expired แล้ว | หากต้องการจองครั้งใหม่ให้เริ่ม search ด้วย key ใหม่ |
| 429 | `RATE_LIMITED` | ส่งคำขอเกินเกณฑ์ที่กำหนด | รอตาม Retry-After และจำกัดจำนวนครั้งที่ลองใหม่ |
| 500 | `INTERNAL_ERROR` | ข้อผิดพลาดภายใน โดยไม่เปิดเผย SQL หรือข้อมูลลับ | ใช้ request_id ให้ทีมตรวจปัญหา ไม่เริ่มการจองใหม่โดยเดาว่างานเดิมล้มเหลว |
| 503 | `TEMPORARILY_UNAVAILABLE` | DB, pool หรือ lock ยังไม่พร้อมรับงานนี้ | รอตาม Retry-After แล้วส่งคำขอเดิมแบบจำกัดจำนวน |
| 504 | `DEADLINE_EXCEEDED` | คำขอใช้เวลาเกินกำหนด อาจ commit แล้วแต่ตอบไม่ทัน | retry confirm ด้วย reservation_id เดิม ไม่สร้างการจองใหม่โดยสรุปเองว่าครั้งแรกไม่สำเร็จ |

429/503 ต้องมี Retry-After เป็นจำนวนวินาทีขั้นต่ำที่แนะนำให้รอ จำกัดจำนวน retries และสุ่มเวลารอเพิ่มเล็กน้อย เพื่อไม่ให้คำขอกลับมาพร้อมกัน หาก network ขาดก่อนมี HTTP response ให้ถือว่ายังไม่รู้ผล ไม่ใช่ยืนยันว่างานล้มเหลว

#### Error response fields

| Field | Type | Mandatory | Nullable | Description |
| --- | --- | --- | --- | --- |
| `error` | object | Yes | No | รหัสและข้อความอธิบายข้อผิดพลาด รวมรายละเอียด field ถ้ามี |
| `error.code` | string | Yes | No | รหัสคงที่สำหรับให้ client เลือกวิธีจัดการร่วมกับ HTTP status ดูรายการในตาราง error codes |
| `error.message` | string | Yes | No | คำอธิบายที่ไม่เปิดเผย SQL, secret หรือ token |
| `error.details` | object[] | No | No | รายละเอียด field ที่ไม่ผ่าน validation เป็น optional field ละได้เมื่อไม่เกี่ยวกับ field และห้ามเป็น null |
| `error.details[].field` | string | Yes เมื่อมีรายการนี้ | No | ชื่อหรือ path ของ field ที่ไม่ถูกต้อง เช่น pattern |
| `error.details[].message` | string | Yes เมื่อมีรายการนี้ | No | คำอธิบายที่อ่านให้ผู้ใช้หรือ developer เข้าใจ ไม่ควรให้ client แยกข้อความนี้เพื่อเลือก logic |
| `request_id` | string (UUID) | Yes | No | UUID สำหรับติดตาม request ที่ไม่สำเร็จด้วย |

#### ตัวอย่าง error — 410 Gone — หมดเวลาจอง

```json
{
  "error": {
    "code": "RESERVATION_EXPIRED",
    "message": "Reservation expired."
  },
  "request_id": "ea10a48c-1b1a-4a3c-8ea8-a8a9d198c539"
}
```

### ข้อควรระวังตอน implement เส้นนี้

ตรวจ DB clock หลังได้ reservation lock การตรวจผ่านก่อน deadline เป็นจุดตัดสินว่าอนุญาต แม้ commit ตามมาหลัง deadline เล็กน้อย ถ้าหมดเวลาต้อง commit การคืนสลากก่อนตอบ 410

## การเปลี่ยนเวอร์ชัน

ใช้ /v1 สำหรับ contract นี้ Request ปฏิเสธ unknown fields ส่วน client ควรอ่าน response ได้แม้ server เพิ่ม field ใหม่ หากเปลี่ยน field บังคับหรือความหมายของ API ต้องมีแผนย้าย client/version ก่อน
