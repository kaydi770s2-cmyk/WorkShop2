



-- =============================================================================
-- CE396 การออกแบบระบบฐานข้อมูล — สัปดาห์ที่ 3 · งาน Workshop 2
-- กรณีศึกษา: ใจดีคลินิกรักษาสัตว์
-- =============================================================================

-- =============================================================================
-- ส่วนที่ 0 — ล้างตารางเก่าก่อนเพื่อให้รันซ้ำได้
-- =============================================================================
DROP TABLE IF EXISTS treatment_visit CASCADE;
DROP TABLE IF EXISTS pet CASCADE;
DROP TABLE IF EXISTS owner CASCADE;


-- =============================================================================
-- ส่วนที่ 1 — เอนทิตี owner (เจ้าของสัตว์เลี้ยง)
-- มาจากแผนภาพ ER: เอนทิตีปกติ (Strong Entity)
-- =============================================================================
CREATE TABLE owner (
    -- คีย์หลัก (Key Attribute): รหัสประจำตัวเจ้าของ สร้างตัวเลขรันไม่อัตโนมัติด้วย SERIAL
    owner_id     SERIAL          PRIMARY KEY,

    -- แอตทริบิวต์ประกอบ (Composite): ชื่อและนามสกุล แยกคอลัมน์เพื่อรองรับการค้นหา
    first_name   VARCHAR(100)    NOT NULL,
    last_name    VARCHAR(100)    NOT NULL,

    -- คีย์คู่แข่ง (Candidate Key / Alternate Key): เลขบัตรประชาชน 13 หลัก ห้ามซ้ำ
    id_card_no   VARCHAR(13)     NOT NULL UNIQUE,

    -- แอตทริบิวต์ประกอบ (Composite): ที่อยู่ เก็บรายละเอียดเพื่อรองรับการจัดกลุ่มตามเขต
    address      TEXT,

    -- ข้อกำหนดระดับตาราง: ป้องกันการกรอกเลขบัตรประชาชนที่ไม่ครบ 13 หลัก
    CONSTRAINT ck_owner_id_card CHECK (LENGTH(id_card_no) = 13)
);

COMMENT ON TABLE owner IS 'ข้อมูลเจ้าของสัตว์เลี้ยง — เอนทิตีปกติ (Strong Entity)';


-- =============================================================================
-- ส่วนที่ 2 — เอนทิตี pet (สัตว์เลี้ยง)
-- มาจากแผนภาพ ER: เอนทิตีปกติ (Strong Entity)
-- =============================================================================
CREATE TABLE pet (
    -- คีย์หลัก (Key Attribute): รหัสสัตว์เลี้ยง
    pet_id       SERIAL          PRIMARY KEY,

    -- คีย์นอก (Foreign Key): เชื่อมโยงกับเจ้าของ ( owner 1 คน มี pet ได้หลายตัว )
    owner_id     INTEGER         NOT NULL REFERENCES owner(owner_id),

    pet_name     VARCHAR(100)    NOT NULL,
    species      VARCHAR(50)     NOT NULL,

    -- วันเกิด: ใช้สำหรับคำนวณอายุสดด้วย AGE(CURRENT_DATE, dob)
    -- *** ไม่มีคอลัมน์ pet_age *** เพราะเป็น Derived Attribute (แอตทริบิวต์สืบทอด)
    dob          DATE,

    gender       CHAR(1)         NOT NULL DEFAULT 'U',

    -- ข้อกำหนดระดับตาราง: บังคับระบุเพศเฉพาะ M (ผู้), F (เมีย), U (ไม่ระบุ)
    CONSTRAINT ck_pet_gender CHECK (gender IN ('M', 'F', 'U'))
);

COMMENT ON TABLE  pet     IS 'ข้อมูลสัตว์เลี้ยง — เอนทิตีปกติ';
COMMENT ON COLUMN pet.dob IS 'วันเกิดของสัตว์เลี้ยง (ใช้คำนวณ Derived Attribute: pet_age)';


-- =============================================================================
-- ส่วนที่ 3 — เอนทิตีอ่อน treatment_visit (ประวัติการรักษา)
-- มาจากแผนภาพ ER: เอนทิตีอ่อน (Weak Entity)
-- =============================================================================
CREATE TABLE treatment_visit (
    -- มาจากเอนทิตีเจ้าของ (Owner Entity): ยืม pet_id มาใช้เป็นส่วนหนึ่งของคีย์หลัก
    pet_id       INTEGER         NOT NULL REFERENCES pet(pet_id),

    -- คีย์บางส่วน (Partial Key / Discriminator): นับลำดับการมารักษา (ครั้งที่ 1, 2, 3...) ของสัตว์แต่ละตัว
    visit_no     INTEGER         NOT NULL,

    -- วันและเวลาเข้ารักษา: บันทึกเวลาปัจจุบันอัตโนมัติด้วย DEFAULT
    visit_date   TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,

    symptoms     TEXT            NOT NULL,
    doctor_name  VARCHAR(100)    NOT NULL,

    -- ยอดเงิน: ใช้ NUMERIC(10,2) เพื่อความแม่นยำทางการเงิน
    total_amount NUMERIC(10,2)   NOT NULL DEFAULT 0.00,
    is_paid      BOOLEAN         NOT NULL DEFAULT FALSE,

    -- คีย์หลักผสม (Composite Primary Key): ผสม (pet_id, visit_no) เพื่อระบุตัวตนของเอนทิตีอ่อน
    CONSTRAINT pk_treatment_visit PRIMARY KEY (pet_id, visit_no),

    -- ข้อกำหนดระดับตาราง: ป้องกันยอดเงินติดลบ และป้องกันการใส่ลำดับครั้งรักษาติดลบ
    CONSTRAINT ck_visit_total_amount CHECK (total_amount >= 0),
    CONSTRAINT ck_visit_no           CHECK (visit_no > 0)
);

COMMENT ON TABLE treatment_visit IS 'ประวัติการเข้ารักษา — เอนทิตีอ่อน (Weak Entity) พึ่งพาตาราง pet';


-- =============================================================================
-- ส่วนที่ 4 — ทดสอบเพิ่มข้อมูล (INSERT)
-- =============================================================================
INSERT INTO owner (first_name, last_name, id_card_no, address) 
VALUES ('กันตภณ', 'กายดี', '1103704001918', '310/357 ถนนสรงประภา เขตดอนเมือง กรุงเทพฯ');

INSERT INTO pet (owner_id, pet_name, species, dob, gender) 
VALUES (1, 'เปอร์เซีย', 'แมว', '2024-04-15', 'F');

INSERT INTO treatment_visit (pet_id, visit_no, symptoms, doctor_name, total_amount, is_paid) 
VALUES 
(1, 1, 'ไข้สูง ซึม ไม่กินอาหาร', 'น.สพ.ใจดี รักสัตว์', 500.00, TRUE),
(1, 2, 'ตรวจติดตามอาการ ฉีดวัคซีน', 'น.สพ.ใจดี รักสัตว์', 350.00, TRUE);


-- =============================================================================
-- ส่วนที่ 5 — ทดลองดึงข้อมูลและคำนวณ Derived Attribute (SELECT)
-- =============================================================================
-- แสดงข้อมูลประวัติการรักษา ร่วมกับการคำนวณอายุสัตว์เลี้ยงสดๆ (AGE)
SELECT 
    p.pet_name AS "ชื่อสัตว์",
    p.species AS "ชนิด",
    AGE(CURRENT_DATE, p.dob) AS "อายุ (Derived Attribute)",
    tv.visit_no AS "รักษาครั้งที่",
    tv.symptoms AS "อาการ",
    tv.total_amount AS "ยอดเงิน"
FROM pet p
JOIN treatment_visit tv ON p.pet_id = tv.pet_id;
