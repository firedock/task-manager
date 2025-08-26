# ADHD Task Manager — Product Spec, Architecture & Starter Code

Designed for fast, low‑friction focus with ADHD‑friendly patterns: tiny decisions, obvious next action, and forgiving offline behavior.

---

## 1) Product Principles

* **One clear next step** on each screen (e.g., *Start Sprint*, *Add Task*).
* **Low stimulation UI**: calm palette, high contrast, generous spacing, large tap targets.
* **Visible momentum**: streaks, progress rings, and trend lines—never zero‑state shame.
* **Fast capture**: single‑field quick add with natural language ("tmr 4pm", "#Corsair @Must").
* **Offline‑first**: everything works without network; background sync when possible.

---

## 2) Core Features → User Stories

* **Task Prioritization**

  * As a user, I set **Priority** (Must/Should/Nice), **Effort** (XS–XL), optional **Deadline**, and the app auto‑ranks tasks.
  * Aging bumps neglected Musts so they resurface.
* **Sprint Timer**

  * Pomodoro‑style sprints (default 25/5) with optional custom presets (15/3, 45/10).
  * Auto‑log time to the active task/project and schedule a local push at sprint end.
* **Progress Check‑ins**

  * Lightweight prompts for **mood**, **energy**, **focus** (3–5 taps total) with optional note.
* **Daily Review**

  * Evening digest (e.g., 8:00 PM local): What got done? What’s top 3 for tomorrow? What blocked you?
* **Time Tracking**

  * Automatic during sprints + manual entries; categorize time (Deep work, Admin, Breaks, etc.).
* **Offline Support**

  * Create/edit everything offline. Mutations queue and reconcile on reconnect with per‑field LWW.

---

## 3) Information Architecture

**Groups** (priority‑ordered) → **Projects** → **Tasks**. Additional entities:

* Users, Devices, SprintSessions, TimeEntries, CheckIns, Habits & HabitLogs, DailyReviews, NotificationPreferences, SyncState.

---

## 4) Prioritization Model (Deterministic & Tunable)

**Score = P + U + A + D − E**

* **P (Priority base)**: Must=60, Should=35, Nice=15
* **U (Urgency)**: if due in `h` hours → `U = clamp(30 - 0.5*h, 0, 30)`; overdue adds +10
* **A (Age)**: days since last workedon → `min(20, 2 * days)`
* **D (Dependency/commitment bonus)**: +10 if part of today’s Top 3; +5 if in active sprint backlog
* **E (Effort penalty)**: XS=0, S=5, M=10, L=18, XL=26

Tie‑breakers: nearest deadline, then shortest effort.

### TypeScript (pure function)

```ts
export type Priority = 'MUST'|'SHOULD'|'NICE';
export type Effort = 'XS'|'S'|'M'|'L'|'XL';

const P_BASE: Record<Priority, number> = { MUST:60, SHOULD:35, NICE:15 };
const E_PEN:  Record<Effort, number>   = { XS:0, S:5, M:10, L:18, XL:26 };

export function taskScore(opts:{
  priority: Priority;
  effort: Effort;
  dueAt?: number;        // ms epoch
  lastWorkedAt?: number; // ms epoch
  isTodayTop3?: boolean;
  inSprintBacklog?: boolean;
  now?: number;
}): number {
  const now = opts.now ?? Date.now();
  let U = 0;
  if (opts.dueAt) {
    const h = Math.max(0, (opts.dueAt - now) / 3.6e6);
    U = Math.max(0, Math.min(30, 30 - 0.5*h));
    if (opts.dueAt < now) U += 10; // overdue bump
  }
  const days = opts.lastWorkedAt ? Math.max(0, (now - opts.lastWorkedAt) / 8.64e7) : 3;
  const A = Math.min(20, 2 * days);
  const D = (opts.isTodayTop3 ? 10 : 0) + (opts.inSprintBacklog ? 5 : 0);
  return Math.round((P_BASE[opts.priority] + U + A + D - E_PEN[opts.effort]) * 10) / 10;
}
```

---

## 5) Data Model (Prisma)

```prisma
// schema.prisma
model User {
  id           String   @id @default(cuid())
  email        String   @unique
  passwordHash String
  pinHash      String?   // optional 4–6 digit PIN (argon2)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  devices      Device[]
  projects     Project[]
}

model Device {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields:[userId], references:[id])
  platform  String   // ios | android
  pushToken String?  // Expo token
  createdAt DateTime @default(now())
  lastSeen  DateTime @default(now())
}

model Group {
  id        String   @id @default(cuid())
  userId    String
  name      String
  priority  Int      // lower = higher in list (0..n)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  projects  Project[]
}

model Project {
  id        String   @id @default(cuid())
  userId    String
  groupId   String
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  tasks     Task[]
}

enum Priority { MUST SHOULD NICE }
enum Effort { XS S M L XL }

model Task {
  id            String   @id @default(cuid())
  userId        String
  projectId     String
  title         String
  notes         String?
  priority      Priority
  effort        Effort
  dueAt         DateTime?
  doneAt        DateTime?
  lastWorkedAt  DateTime?
  inSprintBacklog Boolean @default(false)
  order         Int      // for manual within‑list ordering
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  timeEntries   TimeEntry[]
}

model SprintSession {
  id        String   @id @default(cuid())
  userId    String
  taskId    String?
  projectId String
  startedAt DateTime
  endedAt   DateTime?
  type      String   // sprint | break
  durationS Int      // planned
  createdAt DateTime @default(now())
}

model TimeEntry {
  id        String   @id @default(cuid())
  userId    String
  taskId    String?
  projectId String
  category  String   // deep, admin, meeting, break
  startedAt DateTime
  endedAt   DateTime
  seconds   Int
}

model CheckIn {
  id        String   @id @default(cuid())
  userId    String
  at        DateTime @default(now())
  mood      Int      // 1..5
  energy    Int      // 1..5
  focus     Int      // 1..5
  note      String?
}

model DailyReview {
  id        String   @id @default(cuid())
  userId    String
  date      DateTime // YYYY‑MM‑DD
  done      String[] // list of task ids/titles
  top3      String[] // for tomorrow
  note      String?
  createdAt DateTime @default(now())
}

model Habit {
  id        String   @id @default(cuid())
  userId    String
  name      String
  schedule  String   // cron‑like or RRULE
  createdAt DateTime @default(now())
  logs      HabitLog[]
}

model HabitLog {
  id        String   @id @default(cuid())
  userId    String
  habitId   String
  date      DateTime
  status    String   // done | skipped | partial
}

model SyncShadow {
  id        String   @id @default(cuid())
  userId    String
  entity    String   // table name
  entityId  String
  hash      String   // server hash of last applied state (per field hash optional)
  updatedAt DateTime @updatedAt
}
```

---

## 6) Backend (Node + Express, TypeScript, FP‑style)

**Folder Layout**

```
api/
  src/
    index.ts
    env.ts
    prisma.ts
    http/
      server.ts
      router.ts
      middleware/
        auth.ts
    modules/
      auth.ts
      tasks.ts
      sprints.ts
      checkins.ts
      time.ts
      sync.ts
```

**Auth**

* JWT access (15m) + refresh (14d).
* Optional **PIN** enforced by server (argon2 hash stored on User). App can lock UI locally after N min idle; verify PIN via server when online, or offline by comparing locally cached argon2 hash (pre‑synced).

**Sample: sync (push) minimal**

```ts
// modules/sync.ts
import { Router } from 'express';
import { prisma } from '../prisma';
import { z } from 'zod';

const Mut = z.object({
  entity: z.enum(['Task','Project','Group','CheckIn','TimeEntry','SprintSession','Habit','HabitLog']),
  op: z.enum(['upsert','delete']),
  id: z.string(),
  data: z.record(z.any()).optional(),
  updatedAt: z.string()
});

export const sync = Router().post('/push', async (req, res) => {
  const body = z.object({ mutations: z.array(Mut) }).parse(req.body);
  // idempotency by (entity,id,updatedAt)
  // simple LWW: apply if incoming.updatedAt > server.updatedAt
  const results = [] as any[];
  for (const m of body.mutations) {
    if (m.op === 'delete') {
      await prisma.$executeRawUnsafe(`DELETE FROM "${m.entity}" WHERE id = $1`, m.id);
      results.push({ id: m.id, ok:true });
    } else {
      // use prisma upsert per entity; omitted for brevity
      results.push({ id: m.id, ok:true });
    }
  }
  res.json({ ok:true, results });
});
```

---

## 7) Mobile App (Expo + React Native)

**Structure**

```
app/
  app/(tabs)/index.tsx           // Home (Today)
  app/task/[id].tsx
  app/sprint.tsx
  app/review.tsx
  app/checkin.tsx
  lib/api.ts
  state/store.ts
  state/sprint.ts
  components/TaskCard.tsx
  components/PriorityPills.tsx
  components/TimerRing.tsx
  theme/tokens.ts
```

**State (Zustand, persisted)**

```ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

export const useSprint = create(persist((set,get)=>({
  status: 'idle' as 'idle'|'running'|'break',
  endsAt: 0,
  secondsPlanned: 1500,
  currentTaskId: undefined as string|undefined,
  start(taskId?:string, seconds=1500){
    const endsAt = Date.now() + seconds*1000;
    set({ status:'running', currentTaskId:taskId, endsAt, secondsPlanned: seconds });
  },
  stop(){ set({ status:'idle', endsAt:0, currentTaskId: undefined }); },
}), { name:'sprint', storage: createJSONStorage(()=>AsyncStorage)}));
```

**Scheduling local notifications (Expo)**

```ts
import * as Notifications from 'expo-notifications';

export async function scheduleSprintEnd(endsAt:number){
  const seconds = Math.max(1, Math.round((endsAt - Date.now())/1000));
  return Notifications.scheduleNotificationAsync({
    content: { title: 'Sprint complete', body: 'Nice work — take a short break.' },
    trigger: { seconds },
  });
}
```

**Sprint Timer Screen (sketch)**

```tsx
export default function Sprint() {
  const { status, endsAt, start, stop } = useSprint();
  const remaining = Math.max(0, endsAt - Date.now());
  useEffect(()=>{ if(status==='running') scheduleSprintEnd(endsAt); },[status, endsAt]);
  return (
    <View className="flex-1 items-center justify-center gap-6 p-6 bg-neutral-950">
      <TimerRing ms={remaining} totalMs={useSprint.getState().secondsPlanned*1000} />
      <View className="flex-row gap-3">
        {status!=='running' ? (
          <Button title="Start 25" onPress={()=>start(undefined, 25*60)} />
        ) : (
          <Button title="Stop" onPress={stop} />
        )}
        <Button title="Break 5" onPress={()=>start(undefined, 5*60)} />
      </View>
    </View>
  );
}
```

**Today Screen behaviors**

* Top: *Start Sprint* CTA if idle; otherwise show remaining time.
* "Top 3" horizontal chips.
* Task list sorted by `taskScore()`, group‑separated.
* Quick Add input supports `@Must @Should @Nice`, `!today`, `due: Fri 4p`.

---

## 8) Notifications & Rituals

* **Sprint end warnings**: 60s & 10s chimes optional.
* **Check‑in reminders**: 10:30 AM, 2:30 PM, 6:00 PM local.
* **Daily review**: 8:00 PM local with tomorrow Top‑3 prompt.
* Store per‑device quiet hours; respect system Focus/Do Not Disturb when possible.

---

## 9) Offline‑First Sync (Queue + LWW)

**Client**

* Append mutations: `{entity, op, id, data, updatedAt}` to `mutQueue` in AsyncStorage.
* Apply optimistically to local state.
* Background task periodically POSTs to `/sync/push`; then GET `/sync/pull?since=cursor`.

**Conflicts**

* **Per‑field Last‑Write‑Wins** using ISO `updatedAt` and deviceId; prefer server when equal.
* Soft‑delete via `deletedAt` fields (not shown in schema) for recoverability.

---

## 10) UI / Theme Tokens (calm, high‑contrast)

* **Typography**: Large headings (28/34/700), base (16/24/500).
* **Spacing**: 8‑pt grid; min tap target 44×44.
* **Color**: Background `#0B0F14`, surface `#11161C`, text `#E5ECF2`.
* **Accents**: Must=#FF6B6B, Should=#FFD166, Nice=#4ECDC4. Disabled=#3A4754.
* **Components**: Cards with 16px radius, 1px hairlines, soft elevation.
* **Motion**: 200ms ease‑out; reduce motion option.

---

## 11) API Sketch

```
POST /auth/login          { email, password }
POST /auth/refresh        { refreshToken }
POST /auth/set-pin        { pin }
POST /auth/verify-pin     { pin }
GET  /me

GET  /groups              | POST /groups | PATCH /groups/:id | DELETE /groups/:id
GET  /projects            | POST /projects | ...
GET  /tasks?projectId=    | POST /tasks | PATCH /tasks/:id | DELETE /tasks/:id
POST /sprints/start       { projectId, taskId?, seconds }
POST /sprints/stop        { sessionId }
POST /checkins            { mood, energy, focus, note? }
POST /reviews             { date, done[], top3[], note? }

POST /sync/push           { mutations:[...] }
GET  /sync/pull?since=cursor
```

---

## 12) Security & Privacy

* Hash passwords & PINs with **argon2id**.
* At‑rest encryption on device for sensitive blobs (PIN hash) using secure storage.
* Minimal PII; opt‑in analytics; export/delete my data.

---

## 13) Analytics & Insights (MVP)

* Weekly time breakdown (productive vs non‑productive).
* Focus quality trend vs sprint count.
* Streaks for Daily Reviews and Habits.

---

## 14) Implementation Milestones (fast path)

1. **Week 1**: Schema + Auth + Groups/Projects/Tasks CRUD + prioritization.
2. **Week 2**: Sprint timer + time entries + local notifications.
3. **Week 3**: Check‑ins + Daily review + charts.
4. **Week 4**: Offline queue + sync + PIN lock + polish.

---

## 15) Nice‑to‑Haves (Post‑MVP)

* Calendar view (day/week agenda).
* Web app (Next.js) reusing API.
* Gadget: Home‑screen widgets, WearOS/Watch complications.
* Calendar & Apple/Google Health integrations for energy correlations.

---

### Notes

* iOS background execution is limited; rely on scheduled local notifications and foreground syncs. Use `expo-task-manager` + `expo-background-fetch` conservatively to avoid OS throttling.
* Keep reducers pure and effects idempotent for safe offline replays.
