# ClassOS — Database Schema
## מבנה בסיס הנתונים המלא

---

# 1. דיאגרמת יחסים

```
schools ──< classes ──< class_students >── students
   │            │
   │            └──< planned_lessons ──< delivered_lessons
   │                       │                    │
   │                       │                    └──< progress_logs
   │                       │
   │                       └── curriculum_topics
   │
   └──< school_calendar_events

users ──< user_roles
  │
  └──< teacher_schedules

subjects ──< curricula ──< curriculum_units ──< curriculum_topics

yearly_plans ──< planned_lessons
```

---

# 2. טבלאות

## 2.1 schools — בתי ספר

```sql
CREATE TABLE schools (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,                    -- "בית ספר הגבעה"
  city TEXT,                             -- "תל אביב"
  school_type TEXT DEFAULT 'mamlachti',  -- mamlachti / mamlachti_dati / arab / druze
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

## 2.2 users — משתמשים

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY REFERENCES auth.users(id),
  email TEXT UNIQUE NOT NULL,
  full_name TEXT NOT NULL,               -- "נועה כהן"
  phone TEXT,
  avatar_url TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

## 2.3 user_roles — תפקידים

משתמש יכול להיות גם מורה וגם רכזת באותו בית ספר.

```sql
CREATE TABLE user_roles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  school_id UUID NOT NULL REFERENCES schools(id) ON DELETE CASCADE,
  role TEXT NOT NULL,                    -- teacher / principal / coordinator / content_admin / system_admin
  subject_id UUID REFERENCES subjects(id), -- רק לרכזת מקצוע
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),

  UNIQUE(user_id, school_id, role)
);
```

## 2.4 subjects — מקצועות

```sql
CREATE TABLE subjects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,                    -- "היסטוריה"
  name_en TEXT,                          -- "History"
  icon TEXT,                             -- emoji or icon name
  created_at TIMESTAMPTZ DEFAULT now()
);
```

## 2.5 curricula — תוכניות לימודים (סילבוס)

```sql
CREATE TABLE curricula (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_id UUID NOT NULL REFERENCES subjects(id),
  grade INT NOT NULL,                    -- 7, 8, 9...
  school_type TEXT DEFAULT 'mamlachti',  -- mamlachti / mamlachti_dati
  name TEXT NOT NULL,                    -- "היסטוריה כיתה ז׳ — ממלכתי"
  description TEXT,
  total_recommended_lessons INT,         -- 56
  source_url TEXT,                       -- לינק לפורטל משרד החינוך
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

## 2.6 curriculum_units — יחידות לימוד

```sql
CREATE TABLE curriculum_units (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  curriculum_id UUID NOT NULL REFERENCES curricula(id) ON DELETE CASCADE,
  sort_order INT NOT NULL,               -- 1, 2, 3...
  name TEXT NOT NULL,                    -- "ימי הביניים המוקדמים"
  description TEXT,
  recommended_lessons INT NOT NULL,      -- 14
  has_assessment BOOLEAN DEFAULT true,   -- יש מבחן ביחידה?
  created_at TIMESTAMPTZ DEFAULT now()
);
```

## 2.7 curriculum_topics — נושאים

```sql
CREATE TABLE curriculum_topics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  unit_id UUID NOT NULL REFERENCES curriculum_units(id) ON DELETE CASCADE,
  sort_order INT NOT NULL,               -- 1, 2, 3...
  name TEXT NOT NULL,                    -- "הכנסייה בימי הביניים"
  description TEXT,                      -- תיאור מורחב
  key_concepts TEXT[],                   -- ["היררכיה", "אפיפיור", "כוח דתי"]
  skills TEXT[],                         -- ["השוואה", "ניתוח מקור"]
  recommended_lessons INT DEFAULT 1,     -- כמה שיעורים מומלץ
  textbook_pages TEXT,                   -- "עמ׳ 87-92"
  objectives TEXT[],                     -- מטרות השיעור
  created_at TIMESTAMPTZ DEFAULT now()
);
```

## 2.8 classes — כיתות

```sql
CREATE TABLE classes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id) ON DELETE CASCADE,
  name TEXT NOT NULL,                    -- "ז׳2"
  grade INT NOT NULL,                    -- 7
  academic_year INT NOT NULL,            -- 2027
  level TEXT DEFAULT 'regular',          -- regular / advanced / support
  student_count INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

## 2.9 students — תלמידים

```sql
CREATE TABLE students (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id) ON DELETE CASCADE,
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  student_id_number TEXT,                -- ת.ז. או מספר תלמיד
  gender TEXT,                           -- male / female / other
  notes TEXT,                            -- הערות כלליות
  has_accommodations BOOLEAN DEFAULT false, -- יש התאמות?
  created_at TIMESTAMPTZ DEFAULT now()
);
```

## 2.10 class_students — שיוך תלמידים לכיתות

```sql
CREATE TABLE class_students (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  class_id UUID NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
  student_id UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
  is_active BOOLEAN DEFAULT true,

  UNIQUE(class_id, student_id)
);
```

## 2.11 teacher_schedules — מערכת שעות

```sql
CREATE TABLE teacher_schedules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  class_id UUID NOT NULL REFERENCES classes(id) ON DELETE CASCADE,
  subject_id UUID NOT NULL REFERENCES subjects(id),
  curriculum_id UUID REFERENCES curricula(id),
  day_of_week INT NOT NULL,              -- 0=ראשון, 1=שני... 4=חמישי
  start_time TIME NOT NULL,              -- '10:00'
  end_time TIME NOT NULL,                -- '10:45'
  academic_year INT NOT NULL,            -- 2027
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),

  UNIQUE(user_id, class_id, day_of_week, start_time, academic_year)
);
```

## 2.12 school_calendar_events — לוח שנה בית ספרי

```sql
CREATE TABLE school_calendar_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id) ON DELETE CASCADE,
  name TEXT NOT NULL,                    -- "חופשת סוכות", "יום הזיכרון"
  event_type TEXT NOT NULL,              -- holiday / event / exam_period / special
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  cancels_lessons BOOLEAN DEFAULT true,  -- האם מבטל שיעורים?
  affected_grades INT[],                 -- [7, 8] או NULL = כולם
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

## 2.13 yearly_plans — תוכניות שנתיות

```sql
CREATE TABLE yearly_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  class_id UUID NOT NULL REFERENCES classes(id),
  curriculum_id UUID NOT NULL REFERENCES curricula(id),
  academic_year INT NOT NULL,            -- 2027
  total_available_lessons INT,           -- סה"כ שיעורים זמינים (אחרי חופשות)
  total_planned_lessons INT,             -- סה"כ שיעורים מתוכננים
  reserve_lessons INT,                   -- שיעורי רזרבה
  status TEXT DEFAULT 'draft',           -- draft / active / completed
  generated_at TIMESTAMPTZ,
  approved_by UUID REFERENCES users(id), -- רכזת שאישרה
  approved_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),

  UNIQUE(user_id, class_id, curriculum_id, academic_year)
);
```

## 2.14 planned_lessons — שיעורים מתוכננים

```sql
CREATE TABLE planned_lessons (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  yearly_plan_id UUID NOT NULL REFERENCES yearly_plans(id) ON DELETE CASCADE,
  topic_id UUID REFERENCES curriculum_topics(id),  -- NULL לשיעורי חזרה/הערכה
  lesson_number INT NOT NULL,            -- מספר שיעור בתוכנית (1, 2, 3...)
  lesson_date DATE NOT NULL,
  day_of_week INT NOT NULL,
  start_time TIME NOT NULL,

  lesson_type TEXT DEFAULT 'regular',    -- regular / review / assessment / reserve / makeup
  title TEXT NOT NULL,                   -- "הכנסייה בימי הביניים — עוצמה ותפקיד"

  -- מערך שיעור
  objectives TEXT[],                     -- מטרות
  opening_activity TEXT,                 -- פעילות פתיחה
  main_activity TEXT,                    -- גוף השיעור
  practice_activity TEXT,                -- תרגול
  closing_activity TEXT,                 -- סיכום
  homework TEXT,                         -- שיעורי בית
  materials TEXT[],                      -- חומרים / קישורים
  duration_minutes INT DEFAULT 45,

  -- מטא
  unit_name TEXT,                        -- שם היחידה (לתצוגה מהירה)
  topic_number_in_unit INT,              -- נושא 3 מתוך 7 ביחידה

  status TEXT DEFAULT 'planned',         -- planned / taught / partial / skipped / cancelled
  auto_generated BOOLEAN DEFAULT false,  -- נוצר אוטומטית (חזרה/השלמה)?

  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_planned_lessons_date ON planned_lessons(lesson_date);
CREATE INDEX idx_planned_lessons_plan ON planned_lessons(yearly_plan_id);
```

## 2.15 delivered_lessons — שיעורים שבוצעו (דיווח מורה)

```sql
CREATE TABLE delivered_lessons (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  planned_lesson_id UUID NOT NULL REFERENCES planned_lessons(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES users(id),

  -- דיווח ביצוע
  completion_status TEXT NOT NULL,       -- full / partial / modified / cancelled
  understanding_level TEXT,              -- good / partial / poor
  needs_review BOOLEAN DEFAULT false,

  -- פרטים
  notes TEXT,                            -- הערות חופשיות
  had_significant_absences BOOLEAN DEFAULT false,  -- 5+ חיסורים
  gave_homework BOOLEAN DEFAULT false,
  gave_assessment BOOLEAN DEFAULT false,

  -- מה באמת נלמד (אם שונה מהתוכנית)
  actual_content TEXT,                   -- "לימדתי רק את החלק הראשון"

  reported_at TIMESTAMPTZ DEFAULT now(),
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_delivered_user ON delivered_lessons(user_id);
```

## 2.16 progress_logs — לוג התקדמות (למנועים)

```sql
CREATE TABLE progress_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  class_id UUID NOT NULL REFERENCES classes(id),
  curriculum_id UUID NOT NULL REFERENCES curricula(id),
  unit_id UUID REFERENCES curriculum_units(id),
  topic_id UUID REFERENCES curriculum_topics(id),

  lessons_planned INT DEFAULT 0,
  lessons_delivered INT DEFAULT 0,
  lessons_understood INT DEFAULT 0,      -- "הבינו"
  lessons_need_review INT DEFAULT 0,     -- "צריך חזרה"

  percent_complete DECIMAL(5,2) DEFAULT 0,

  last_updated TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_progress_class ON progress_logs(class_id, curriculum_id);
```

---

# 3. RLS Policies — הרשאות

```sql
-- מורה רואה רק את הכיתות שלה
CREATE POLICY "teachers see own classes" ON classes
  FOR SELECT USING (
    id IN (
      SELECT class_id FROM teacher_schedules
      WHERE user_id = auth.uid() AND is_active = true
    )
  );

-- מנהלת רואה את כל הכיתות בבית הספר שלה
CREATE POLICY "principals see school classes" ON classes
  FOR SELECT USING (
    school_id IN (
      SELECT school_id FROM user_roles
      WHERE user_id = auth.uid() AND role = 'principal'
    )
  );

-- מורה רואה רק את השיעורים שלה
CREATE POLICY "teachers see own lessons" ON planned_lessons
  FOR SELECT USING (
    yearly_plan_id IN (
      SELECT id FROM yearly_plans WHERE user_id = auth.uid()
    )
  );

-- מורה יכולה לדווח רק על השיעורים שלה
CREATE POLICY "teachers report own lessons" ON delivered_lessons
  FOR INSERT WITH CHECK (user_id = auth.uid());
```

---

# 4. Seed Data — נתוני בסיס

```sql
-- מקצוע
INSERT INTO subjects (name, name_en, icon)
VALUES ('היסטוריה', 'History', '📜');

-- סילבוס
INSERT INTO curricula (subject_id, grade, school_type, name, total_recommended_lessons)
VALUES ([history_id], 7, 'mamlachti', 'היסטוריה כיתה ז׳ — ממלכתי', 56);

-- לוח שנה ארצי 2026-2027 (חופשות עיקריות)
INSERT INTO school_calendar_events (school_id, name, event_type, start_date, end_date) VALUES
([school], 'חופשת סוכות', 'holiday', '2026-10-02', '2026-10-11'),
([school], 'חופשת חנוכה', 'holiday', '2026-12-25', '2027-01-01'),
([school], 'חופשת פורים', 'holiday', '2027-03-12', '2027-03-14'),
([school], 'חופשת פסח', 'holiday', '2027-03-31', '2027-04-13'),
([school], 'יום הזיכרון', 'holiday', '2027-04-21', '2027-04-21'),
([school], 'יום העצמאות', 'holiday', '2027-04-22', '2027-04-22'),
([school], 'שבועות', 'holiday', '2027-05-21', '2027-05-21');
```

---

# 5. סדר יצירת טבלאות (Migration Order)

```
1. schools
2. subjects
3. users
4. user_roles
5. curricula
6. curriculum_units
7. curriculum_topics
8. classes
9. students
10. class_students
11. teacher_schedules
12. school_calendar_events
13. yearly_plans
14. planned_lessons
15. delivered_lessons
16. progress_logs
```

---

# 6. אינדקסים קריטיים

```sql
-- חיפוש שיעורים לפי תאריך (תצוגת שבוע)
CREATE INDEX idx_lessons_date_plan ON planned_lessons(yearly_plan_id, lesson_date);

-- חיפוש דיווחים לפי מורה (דשבורד)
CREATE INDEX idx_delivered_user_date ON delivered_lessons(user_id, reported_at);

-- התקדמות לפי כיתה (דשבורד מנהלת)
CREATE INDEX idx_progress_class_curriculum ON progress_logs(class_id, curriculum_id);

-- מערכת שעות פעילה
CREATE INDEX idx_schedules_active ON teacher_schedules(user_id, academic_year) WHERE is_active = true;
```
