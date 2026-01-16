# School-Schedule
import React, { useEffect, useMemo, useState } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Textarea } from "@/components/ui/textarea";
import {
  Dialog,
  DialogContent,
  DialogFooter,
  DialogHeader,
  DialogTitle,
  DialogTrigger,
} from "@/components/ui/dialog";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import {
  Tabs,
  TabsContent,
  TabsList,
  TabsTrigger,
} from "@/components/ui/tabs";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { Badge } from "@/components/ui/badge";
import { Checkbox } from "@/components/ui/checkbox";
import { CalendarDays, CheckCircle2, Dumbbell, GraduationCap, Plus, Trash2 } from "lucide-react";

// -----------------------------
// Minimal student planner website
// - Classes: weekly schedule blocks
// - School work: assignments & tasks
// - Athletics: practices & games
// - Calendar-like day view: upcoming items
// - Local persistence via localStorage
// -----------------------------

const LS_KEY = "student_planner_v1";

function uid() {
  return Math.random().toString(36).slice(2) + Date.now().toString(36);
}

function todayISO() {
  const d = new Date();
  const yyyy = d.getFullYear();
  const mm = String(d.getMonth() + 1).padStart(2, "0");
  const dd = String(d.getDate()).padStart(2, "0");
  return `${yyyy}-${mm}-${dd}`;
}

function addDaysISO(iso, days) {
  const d = new Date(iso + "T00:00:00");
  d.setDate(d.getDate() + days);
  const yyyy = d.getFullYear();
  const mm = String(d.getMonth() + 1).padStart(2, "0");
  const dd = String(d.getDate()).padStart(2, "0");
  return `${yyyy}-${mm}-${dd}`;
}

function weekdayName(idx) {
  return ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"][idx];
}

function timeToMinutes(t) {
  // "HH:MM" -> minutes
  const [h, m] = t.split(":").map((x) => parseInt(x, 10));
  return h * 60 + m;
}

function fmtDate(iso) {
  const d = new Date(iso + "T00:00:00");
  return d.toLocaleDateString(undefined, { weekday: "short", month: "short", day: "numeric" });
}

function priorityBadge(p) {
  if (p === "High") return <Badge className="bg-red-600">High</Badge>;
  if (p === "Medium") return <Badge className="bg-amber-600">Medium</Badge>;
  return <Badge className="bg-slate-600">Low</Badge>;
}

const seed = {
  profile: { name: "Student", school: "Your School", sport: "Your Sport" },
  classes: [
    { id: uid(), name: "Math", day: 0, start: "08:00", end: "08:55", location: "Room 201" },
    { id: uid(), name: "English", day: 0, start: "09:05", end: "10:00", location: "Room 105" },
    { id: uid(), name: "Chem", day: 1, start: "08:00", end: "08:55", location: "Lab A" },
    { id: uid(), name: "History", day: 2, start: "10:10", end: "11:05", location: "Room 312" },
  ],
  tasks: [
    {
      id: uid(),
      title: "Finish math problem set",
      type: "School Work",
      due: addDaysISO(todayISO(), 2),
      priority: "High",
      done: false,
      notes: "Do #1-20. Check answers.",
    },
    {
      id: uid(),
      title: "Read chapter for English",
      type: "School Work",
      due: addDaysISO(todayISO(), 1),
      priority: "Medium",
      done: false,
      notes: "Annotate 5 quotes.",
    },
    {
      id: uid(),
      title: "Foam roll + stretch",
      type: "Athletics",
      due: addDaysISO(todayISO(), 0),
      priority: "Low",
      done: false,
      notes: "10 min calves, quads, glutes.",
    },
  ],
  athletics: [
    { id: uid(), title: "Practice", date: addDaysISO(todayISO(), 0), start: "16:00", end: "18:00", location: "Field" },
    { id: uid(), title: "Game vs Rival", date: addDaysISO(todayISO(), 3), start: "17:30", end: "19:00", location: "Stadium" },
  ],
};

function loadState() {
  try {
    const raw = localStorage.getItem(LS_KEY);
    if (!raw) return seed;
    const parsed = JSON.parse(raw);
    // Light validation
    if (!parsed || !parsed.profile || !Array.isArray(parsed.classes) || !Array.isArray(parsed.tasks) || !Array.isArray(parsed.athletics)) {
      return seed;
    }
    return parsed;
  } catch {
    return seed;
  }
}

function saveState(state) {
  try {
    localStorage.setItem(LS_KEY, JSON.stringify(state));
  } catch {
    // ignore
  }
}

function SectionTitle({ icon, title, subtitle }) {
  return (
    <div className="flex items-start justify-between gap-3">
      <div className="flex items-start gap-3">
        <div className="mt-1 text-slate-700">{icon}</div>
        <div>
          <div className="text-lg font-semibold text-slate-900">{title}</div>
          {subtitle ? <div className="text-sm text-slate-500">{subtitle}</div> : null}
        </div>
      </div>
    </div>
  );
}

function EmptyState({ title, hint }) {
  return (
    <div className="rounded-2xl border border-dashed p-6 text-center">
      <div className="font-medium text-slate-800">{title}</div>
      {hint ? <div className="mt-1 text-sm text-slate-500">{hint}</div> : null}
    </div>
  );
}

function AddClassDialog({ onAdd }) {
  const [open, setOpen] = useState(false);
  const [name, setName] = useState("");
  const [day, setDay] = useState("0");
  const [start, setStart] = useState("08:00");
  const [end, setEnd] = useState("09:00");
  const [location, setLocation] = useState("");

  const canSave = name.trim() && start < end;

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button className="rounded-2xl"><Plus className="mr-2 h-4 w-4" />Add class</Button>
      </DialogTrigger>
      <DialogContent className="sm:max-w-[520px]">
        <DialogHeader>
          <DialogTitle>Add a class</DialogTitle>
        </DialogHeader>
        <div className="grid gap-4">
          <div className="grid gap-2">
            <Label>Class name</Label>
            <Input value={name} onChange={(e) => setName(e.target.value)} placeholder="e.g., AP Biology" />
          </div>
          <div className="grid grid-cols-2 gap-3">
            <div className="grid gap-2">
              <Label>Day</Label>
              <Select value={day} onValueChange={setDay}>
                <SelectTrigger><SelectValue placeholder="Choose" /></SelectTrigger>
                <SelectContent>
                  {[0,1,2,3,4,5,6].map((d) => (
                    <SelectItem key={d} value={String(d)}>{weekdayName(d)}</SelectItem>
                  ))}
                </SelectContent>
              </Select>
            </div>
            <div className="grid gap-2">
              <Label>Location</Label>
              <Input value={location} onChange={(e) => setLocation(e.target.value)} placeholder="Room / building" />
            </div>
          </div>
          <div className="grid grid-cols-2 gap-3">
            <div className="grid gap-2">
              <Label>Start</Label>
              <Input type="time" value={start} onChange={(e) => setStart(e.target.value)} />
            </div>
            <div className="grid gap-2">
              <Label>End</Label>
              <Input type="time" value={end} onChange={(e) => setEnd(e.target.value)} />
            </div>
          </div>
        </div>
        <DialogFooter>
          <Button variant="secondary" className="rounded-2xl" onClick={() => setOpen(false)}>Cancel</Button>
          <Button
            className="rounded-2xl"
            disabled={!canSave}
            onClick={() => {
              onAdd({ id: uid(), name: name.trim(), day: parseInt(day, 10), start, end, location: location.trim() });
              setName("");
              setLocation("");
              setDay("0");
              setStart("08:00");
              setEnd("09:00");
              setOpen(false);
            }}
          >
            Save
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}

function AddTaskDialog({ onAdd }) {
  const [open, setOpen] = useState(false);
  const [title, setTitle] = useState("");
  const [type, setType] = useState("School Work");
  const [due, setDue] = useState(todayISO());
  const [priority, setPriority] = useState("Medium");
  const [notes, setNotes] = useState("");
  const canSave = title.trim();

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button className="rounded-2xl"><Plus className="mr-2 h-4 w-4" />Add task</Button>
      </DialogTrigger>
      <DialogContent className="sm:max-w-[560px]">
        <DialogHeader>
          <DialogTitle>Add a task</DialogTitle>
        </DialogHeader>
        <div className="grid gap-4">
          <div className="grid gap-2">
            <Label>Title</Label>
            <Input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="e.g., Essay draft" />
          </div>
          <div className="grid grid-cols-2 gap-3">
            <div className="grid gap-2">
              <Label>Category</Label>
              <Select value={type} onValueChange={setType}>
                <SelectTrigger><SelectValue /></SelectTrigger>
                <SelectContent>
                  <SelectItem value="School Work">School Work</SelectItem>
                  <SelectItem value="Athletics">Athletics</SelectItem>
                  <SelectItem value="Personal">Personal</SelectItem>
                </SelectContent>
              </Select>
            </div>
            <div className="grid gap-2">
              <Label>Due date</Label>
              <Input type="date" value={due} onChange={(e) => setDue(e.target.value)} />
            </div>
          </div>
          <div className="grid grid-cols-2 gap-3">
            <div className="grid gap-2">
              <Label>Priority</Label>
              <Select value={priority} onValueChange={setPriority}>
                <SelectTrigger><SelectValue /></SelectTrigger>
                <SelectContent>
                  <SelectItem value="High">High</SelectItem>
                  <SelectItem value="Medium">Medium</SelectItem>
                  <SelectItem value="Low">Low</SelectItem>
                </SelectContent>
              </Select>
            </div>
            <div className="grid gap-2">
              <Label>Notes</Label>
              <Input value={notes} onChange={(e) => setNotes(e.target.value)} placeholder="Optional" />
            </div>
          </div>
          <div className="grid gap-2">
            <Label>Details</Label>
            <Textarea value={notes} onChange={(e) => setNotes(e.target.value)} placeholder="What to do, links, checklist, etc." />
          </div>
        </div>
        <DialogFooter>
          <Button variant="secondary" className="rounded-2xl" onClick={() => setOpen(false)}>Cancel</Button>
          <Button
            className="rounded-2xl"
            disabled={!canSave}
            onClick={() => {
              onAdd({ id: uid(), title: title.trim(), type, due, priority, done: false, notes });
              setTitle("");
              setType("School Work");
              setDue(todayISO());
              setPriority("Medium");
              setNotes("");
              setOpen(false);
            }}
          >
            Save
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}

function AddAthleticsDialog({ onAdd }) {
  const [open, setOpen] = useState(false);
  const [title, setTitle] = useState("Practice");
  const [date, setDate] = useState(todayISO());
  const [start, setStart] = useState("16:00");
  const [end, setEnd] = useState("18:00");
  const [location, setLocation] = useState("Field");
  const canSave = title.trim() && start < end;

  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogTrigger asChild>
        <Button className="rounded-2xl"><Plus className="mr-2 h-4 w-4" />Add athletics</Button>
      </DialogTrigger>
      <DialogContent className="sm:max-w-[560px]">
        <DialogHeader>
          <DialogTitle>Add athletics event</DialogTitle>
        </DialogHeader>
        <div className="grid gap-4">
          <div className="grid gap-2">
            <Label>Event</Label>
            <Input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="Practice, game, lift, etc." />
          </div>
          <div className="grid grid-cols-2 gap-3">
            <div className="grid gap-2">
              <Label>Date</Label>
              <Input type="date" value={date} onChange={(e) => setDate(e.target.value)} />
            </div>
            <div className="grid gap-2">
              <Label>Location</Label>
              <Input value={location} onChange={(e) => setLocation(e.target.value)} />
            </div>
          </div>
          <div className="grid grid-cols-2 gap-3">
            <div className="grid gap-2">
              <Label>Start</Label>
              <Input type="time" value={start} onChange={(e) => setStart(e.target.value)} />
            </div>
            <div className="grid gap-2">
              <Label>End</Label>
              <Input type="time" value={end} onChange={(e) => setEnd(e.target.value)} />
            </div>
          </div>
        </div>
        <DialogFooter>
          <Button variant="secondary" className="rounded-2xl" onClick={() => setOpen(false)}>Cancel</Button>
          <Button
            className="rounded-2xl"
            disabled={!canSave}
            onClick={() => {
              onAdd({ id: uid(), title: title.trim(), date, start, end, location: location.trim() });
              setOpen(false);
            }}
          >
            Save
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}

function UpcomingCard({ title, items, emptyHint }) {
  return (
    <Card className="rounded-3xl shadow-sm">
      <CardHeader className="pb-2">
        <CardTitle className="text-base">{title}</CardTitle>
      </CardHeader>
      <CardContent className="pt-0">
        {items.length === 0 ? (
          <div className="text-sm text-slate-500">{emptyHint}</div>
        ) : (
          <div className="space-y-2">
            {items.slice(0, 6).map((it) => (
              <div key={it.key} className="flex items-start justify-between gap-3 rounded-2xl border p-3">
                <div className="min-w-0">
                  <div className="truncate font-medium text-slate-900">{it.primary}</div>
                  <div className="truncate text-xs text-slate-500">{it.secondary}</div>
                </div>
                <div className="shrink-0 text-xs text-slate-600">{it.right}</div>
              </div>
            ))}
          </div>
        )}
      </CardContent>
    </Card>
  );
}

export default function StudentPlannerWebsite() {
  const [state, setState] = useState(loadState);
  const [taskFilter, setTaskFilter] = useState("All");
  const [search, setSearch] = useState("");

  useEffect(() => {
    saveState(state);
  }, [state]);

  const upcoming = useMemo(() => {
    const now = todayISO();
    const horizon = addDaysISO(now, 7);

    const tasks = state.tasks
      .filter((t) => !t.done)
      .filter((t) => t.due >= now && t.due <= horizon)
      .sort((a, b) => (a.due === b.due ? (a.priority < b.priority ? 1 : -1) : a.due.localeCompare(b.due)))
      .map((t) => ({
        key: t.id,
        primary: t.title,
        secondary: `${t.type} • due ${fmtDate(t.due)}`,
        right: t.priority,
      }));

    const athletics = state.athletics
      .filter((e) => e.date >= now && e.date <= horizon)
      .sort((a, b) => (a.date === b.date ? a.start.localeCompare(b.start) : a.date.localeCompare(b.date)))
      .map((e) => ({
        key: e.id,
        primary: e.title,
        secondary: `${fmtDate(e.date)} • ${e.location || ""}`,
        right: `${e.start}-${e.end}`,
      }));

    return { tasks, athletics };
  }, [state]);

  const weekSchedule = useMemo(() => {
    const byDay = Array.from({ length: 7 }, () => []);
    for (const c of state.classes) byDay[c.day].push(c);
    for (const d of byDay) d.sort((a, b) => timeToMinutes(a.start) - timeToMinutes(b.start));
    return byDay;
  }, [state]);

  const filteredTasks = useMemo(() => {
    const q = search.trim().toLowerCase();
    return state.tasks
      .filter((t) => (taskFilter === "All" ? true : t.type === taskFilter))
      .filter((t) => (q ? (t.title + " " + (t.notes || "")).toLowerCase().includes(q) : true))
      .sort((a, b) => {
        if (a.done !== b.done) return a.done ? 1 : -1;
        if (a.due !== b.due) return a.due.localeCompare(b.due);
        return a.priority === b.priority ? 0 : a.priority === "High" ? -1 : a.priority === "Low" ? 1 : 0;
      });
  }, [state.tasks, taskFilter, search]);

  const dayView = useMemo(() => {
    // Combines classes, tasks, athletics for a selected day
    return function build(iso) {
      const d = new Date(iso + "T00:00:00");
      const jsDay = d.getDay(); // 0 Sun..6 Sat
      const ourDay = (jsDay + 6) % 7; // 0 Mon..6 Sun

      const classes = state.classes
        .filter((c) => c.day === ourDay)
        .sort((a, b) => timeToMinutes(a.start) - timeToMinutes(b.start))
        .map((c) => ({
          key: c.id,
          time: `${c.start}-${c.end}`,
          title: c.name,
          meta: c.location || "",
          kind: "Class",
        }));

      const athletics = state.athletics
        .filter((e) => e.date === iso)
        .sort((a, b) => a.start.localeCompare(b.start))
        .map((e) => ({
          key: e.id,
          time: `${e.start}-${e.end}`,
          title: e.title,
          meta: e.location || "",
          kind: "Athletics",
        }));

      const tasksDue = state.tasks
        .filter((t) => t.due === iso)
        .sort((a, b) => (a.done !== b.done ? (a.done ? 1 : -1) : 0))
        .map((t) => ({
          key: t.id,
          time: "All day",
          title: t.title,
          meta: `${t.type} • ${t.priority}`,
          kind: "Task",
        }));

      return [...classes, ...athletics, ...tasksDue].sort((a, b) => {
        if (a.time === "All day" && b.time !== "All day") return 1;
        if (b.time === "All day" && a.time !== "All day") return -1;
        return a.time.localeCompare(b.time);
      });
    };
  }, [state]);

  const [selectedDate, setSelectedDate] = useState(todayISO());
  const selectedDayItems = useMemo(() => dayView(selectedDate), [dayView, selectedDate]);

  const stats = useMemo(() => {
    const open = state.tasks.filter((t) => !t.done).length;
    const done = state.tasks.filter((t) => t.done).length;
    const nextPractice = upcoming.athletics[0];
    return { open, done, nextPractice };
  }, [state.tasks, upcoming.athletics]);

  return (
    <div className="min-h-screen bg-gradient-to-b from-slate-50 to-white">
      <div className="mx-auto max-w-6xl px-4 py-6">
        <header className="flex flex-col gap-3 md:flex-row md:items-center md:justify-between">
          <div className="space-y-1">
            <div className="text-2xl font-semibold text-slate-900">Student Planner</div>
            <div className="text-sm text-slate-600">
              {state.profile.name} • {state.profile.school} • {state.profile.sport}
            </div>
          </div>
          <div className="flex flex-wrap gap-2">
            <Button
              variant="secondary"
              className="rounded-2xl"
              onClick={() => {
                setState(seed);
              }}
              title="Resets to sample data"
            >
              Reset sample
            </Button>
            <Button
              variant="outline"
              className="rounded-2xl"
              onClick={() => {
                localStorage.removeItem(LS_KEY);
                setState(seed);
              }}
              title="Clears saved data"
            >
              Clear saved
            </Button>
          </div>
        </header>

        <div className="mt-6">
          <Tabs defaultValue="dashboard">
            <TabsList className="grid w-full grid-cols-2 rounded-2xl md:w-auto md:grid-cols-5">
              <TabsTrigger value="dashboard" className="rounded-2xl">Dashboard</TabsTrigger>
              <TabsTrigger value="calendar" className="rounded-2xl">Day View</TabsTrigger>
              <TabsTrigger value="classes" className="rounded-2xl">Classes</TabsTrigger>
              <TabsTrigger value="tasks" className="rounded-2xl">Tasks</TabsTrigger>
              <TabsTrigger value="athletics" className="rounded-2xl">Athletics</TabsTrigger>
            </TabsList>

            {/* Dashboard */}
            <TabsContent value="dashboard" className="mt-5">
              <div className="grid gap-4 md:grid-cols-3">
                <Card className="rounded-3xl shadow-sm">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Today</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    <div className="grid gap-3">
                      <div className="flex items-center justify-between rounded-2xl border p-3">
                        <div className="flex items-center gap-2 text-slate-700">
                          <CheckCircle2 className="h-4 w-4" />
                          <span className="text-sm">Open tasks</span>
                        </div>
                        <div className="font-semibold text-slate-900">{stats.open}</div>
                      </div>
                      <div className="flex items-center justify-between rounded-2xl border p-3">
                        <div className="flex items-center gap-2 text-slate-700">
                          <CheckCircle2 className="h-4 w-4" />
                          <span className="text-sm">Completed</span>
                        </div>
                        <div className="font-semibold text-slate-900">{stats.done}</div>
                      </div>
                      <div className="rounded-2xl border p-3">
                        <div className="text-xs text-slate-500">Next athletics</div>
                        {stats.nextPractice ? (
                          <div className="mt-1 text-sm font-medium text-slate-900">
                            {stats.nextPractice.primary}
                            <div className="text-xs font-normal text-slate-500">{stats.nextPractice.secondary}</div>
                          </div>
                        ) : (
                          <div className="mt-1 text-sm text-slate-600">None in next 7 days</div>
                        )}
                      </div>
                    </div>
                  </CardContent>
                </Card>

                <UpcomingCard
                  title="Upcoming tasks (7 days)"
                  items={upcoming.tasks.map((t) => ({ ...t, right: t.right }))}
                  emptyHint="No tasks due in the next 7 days."
                />

                <UpcomingCard
                  title="Athletics (7 days)"
                  items={upcoming.athletics}
                  emptyHint="No practices or games in the next 7 days."
                />
              </div>

              <div className="mt-4 grid gap-4 md:grid-cols-2">
                <Card className="rounded-3xl shadow-sm">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Weekly class schedule</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    <div className="grid gap-3 md:grid-cols-2">
                      {[0,1,2,3,4,5,6].map((d) => (
                        <div key={d} className="rounded-2xl border p-3">
                          <div className="mb-2 text-sm font-semibold text-slate-900">{weekdayName(d)}</div>
                          {weekSchedule[d].length === 0 ? (
                            <div className="text-xs text-slate-500">No classes</div>
                          ) : (
                            <div className="space-y-2">
                              {weekSchedule[d].slice(0, 3).map((c) => (
                                <div key={c.id} className="rounded-xl bg-slate-50 p-2">
                                  <div className="text-sm font-medium text-slate-900">{c.name}</div>
                                  <div className="text-xs text-slate-500">{c.start}-{c.end}{c.location ? ` • ${c.location}` : ""}</div>
                                </div>
                              ))}
                              {weekSchedule[d].length > 3 ? (
                                <div className="text-xs text-slate-500">+{weekSchedule[d].length - 3} more</div>
                              ) : null}
                            </div>
                          )}
                        </div>
                      ))}
                    </div>
                  </CardContent>
                </Card>

                <Card className="rounded-3xl shadow-sm">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Quick add</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    <div className="grid gap-3">
                      <div className="flex flex-wrap gap-2">
                        <AddClassDialog
                          onAdd={(cls) => setState((s) => ({ ...s, classes: [...s.classes, cls] }))}
                        />
                        <AddTaskDialog
                          onAdd={(task) => setState((s) => ({ ...s, tasks: [...s.tasks, task] }))}
                        />
                        <AddAthleticsDialog
                          onAdd={(evt) => setState((s) => ({ ...s, athletics: [...s.athletics, evt] }))}
                        />
                      </div>
                      <div className="text-sm text-slate-600">
                        Use this site for one thing: keep school and athletics in one place.
                      </div>
                      <div className="text-xs text-slate-500">
                        Data saves automatically in your browser.
                      </div>
                    </div>
                  </CardContent>
                </Card>
              </div>
            </TabsContent>

            {/* Day View */}
            <TabsContent value="calendar" className="mt-5">
              <div className="grid gap-4 md:grid-cols-3">
                <Card className="rounded-3xl shadow-sm md:col-span-1">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Pick a date</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    <div className="grid gap-3">
                      <div className="grid gap-2">
                        <Label>Date</Label>
                        <Input type="date" value={selectedDate} onChange={(e) => setSelectedDate(e.target.value)} />
                      </div>
                      <div className="grid grid-cols-3 gap-2">
                        <Button variant="outline" className="rounded-2xl" onClick={() => setSelectedDate(addDaysISO(selectedDate, -1))}>-1</Button>
                        <Button variant="outline" className="rounded-2xl" onClick={() => setSelectedDate(todayISO())}>Today</Button>
                        <Button variant="outline" className="rounded-2xl" onClick={() => setSelectedDate(addDaysISO(selectedDate, 1))}>+1</Button>
                      </div>
                      <div className="rounded-2xl border p-3 text-sm text-slate-700">
                        <div className="flex items-center gap-2">
                          <CalendarDays className="h-4 w-4" />
                          <span className="font-medium">{fmtDate(selectedDate)}</span>
                        </div>
                        <div className="mt-1 text-xs text-slate-500">Classes repeat weekly. Tasks and athletics are dated.</div>
                      </div>
                    </div>
                  </CardContent>
                </Card>

                <Card className="rounded-3xl shadow-sm md:col-span-2">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Schedule for {fmtDate(selectedDate)}</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    {selectedDayItems.length === 0 ? (
                      <EmptyState title="Nothing scheduled" hint="Add a task, class, or athletics event." />
                    ) : (
                      <div className="space-y-2">
                        {selectedDayItems.map((it) => (
                          <div key={it.key} className="flex items-start justify-between gap-3 rounded-2xl border p-3">
                            <div className="min-w-0">
                              <div className="flex items-center gap-2">
                                <Badge variant="secondary" className="rounded-xl">{it.kind}</Badge>
                                <div className="truncate font-medium text-slate-900">{it.title}</div>
                              </div>
                              {it.meta ? <div className="mt-1 truncate text-xs text-slate-500">{it.meta}</div> : null}
                            </div>
                            <div className="shrink-0 text-sm text-slate-700">{it.time}</div>
                          </div>
                        ))}
                      </div>
                    )}
                  </CardContent>
                </Card>
              </div>
            </TabsContent>

            {/* Classes */}
            <TabsContent value="classes" className="mt-5">
              <div className="grid gap-4 md:grid-cols-3">
                <Card className="rounded-3xl shadow-sm md:col-span-1">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Add</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    <SectionTitle
                      icon={<GraduationCap className="h-4 w-4" />}
                      title="Classes"
                      subtitle="Weekly repeating schedule"
                    />
                    <div className="mt-3">
                      <AddClassDialog onAdd={(cls) => setState((s) => ({ ...s, classes: [...s.classes, cls] }))} />
                    </div>
                    <div className="mt-3 text-xs text-slate-500">
                      Tip: keep locations short. Example: “Rm 201”.
                    </div>
                  </CardContent>
                </Card>

                <Card className="rounded-3xl shadow-sm md:col-span-2">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Weekly schedule</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    {state.classes.length === 0 ? (
                      <EmptyState title="No classes yet" hint="Add your first class." />
                    ) : (
                      <Table>
                        <TableHeader>
                          <TableRow>
                            <TableHead>Day</TableHead>
                            <TableHead>Time</TableHead>
                            <TableHead>Class</TableHead>
                            <TableHead>Location</TableHead>
                            <TableHead className="text-right">Actions</TableHead>
                          </TableRow>
                        </TableHeader>
                        <TableBody>
                          {state.classes
                            .slice()
                            .sort((a, b) => (a.day !== b.day ? a.day - b.day : timeToMinutes(a.start) - timeToMinutes(b.start)))
                            .map((c) => (
                              <TableRow key={c.id}>
                                <TableCell className="font-medium">{weekdayName(c.day)}</TableCell>
                                <TableCell>{c.start}–{c.end}</TableCell>
                                <TableCell>{c.name}</TableCell>
                                <TableCell className="text-slate-600">{c.location || ""}</TableCell>
                                <TableCell className="text-right">
                                  <Button
                                    variant="outline"
                                    size="sm"
                                    className="rounded-xl"
                                    onClick={() => setState((s) => ({ ...s, classes: s.classes.filter((x) => x.id !== c.id) }))}
                                  >
                                    <Trash2 className="h-4 w-4" />
                                  </Button>
                                </TableCell>
                              </TableRow>
                            ))}
                        </TableBody>
                      </Table>
                    )}
                  </CardContent>
                </Card>
              </div>
            </TabsContent>

            {/* Tasks */}
            <TabsContent value="tasks" className="mt-5">
              <div className="grid gap-4 md:grid-cols-3">
                <Card className="rounded-3xl shadow-sm md:col-span-1">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Controls</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    <SectionTitle
                      icon={<CheckCircle2 className="h-4 w-4" />}
                      title="Tasks"
                      subtitle="Homework, training, personal"
                    />
                    <div className="mt-3 grid gap-3">
                      <AddTaskDialog onAdd={(task) => setState((s) => ({ ...s, tasks: [...s.tasks, task] }))} />

                      <div className="grid gap-2">
                        <Label>Filter</Label>
                        <Select value={taskFilter} onValueChange={setTaskFilter}>
                          <SelectTrigger className="rounded-2xl"><SelectValue /></SelectTrigger>
                          <SelectContent>
                            <SelectItem value="All">All</SelectItem>
                            <SelectItem value="School Work">School Work</SelectItem>
                            <SelectItem value="Athletics">Athletics</SelectItem>
                            <SelectItem value="Personal">Personal</SelectItem>
                          </SelectContent>
                        </Select>
                      </div>

                      <div className="grid gap-2">
                        <Label>Search</Label>
                        <Input value={search} onChange={(e) => setSearch(e.target.value)} placeholder="Find tasks" />
                      </div>

                      <div className="rounded-2xl border p-3 text-xs text-slate-500">
                        Fast workflow: add tasks, check them off, delete later.
                      </div>
                    </div>
                  </CardContent>
                </Card>

                <Card className="rounded-3xl shadow-sm md:col-span-2">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Task list</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    {filteredTasks.length === 0 ? (
                      <EmptyState title="No tasks match" hint="Change filter or add a task." />
                    ) : (
                      <div className="space-y-2">
                        {filteredTasks.map((t) => (
                          <div key={t.id} className="flex items-start justify-between gap-3 rounded-2xl border p-3">
                            <div className="flex items-start gap-3">
                              <Checkbox
                                checked={t.done}
                                onCheckedChange={(v) =>
                                  setState((s) => ({
                                    ...s,
                                    tasks: s.tasks.map((x) => (x.id === t.id ? { ...x, done: Boolean(v) } : x)),
                                  }))
                                }
                                className="mt-1"
                              />
                              <div className="min-w-0">
                                <div className={`truncate font-medium ${t.done ? "text-slate-400 line-through" : "text-slate-900"}`}>{t.title}</div>
                                <div className="mt-1 flex flex-wrap items-center gap-2 text-xs text-slate-500">
                                  <Badge variant="secondary" className="rounded-xl">{t.type}</Badge>
                                  <span>Due {fmtDate(t.due)}</span>
                                  {priorityBadge(t.priority)}
                                </div>
                                {t.notes ? <div className="mt-2 text-sm text-slate-700">{t.notes}</div> : null}
                              </div>
                            </div>
                            <div className="shrink-0">
                              <Button
                                variant="outline"
                                size="sm"
                                className="rounded-xl"
                                onClick={() => setState((s) => ({ ...s, tasks: s.tasks.filter((x) => x.id !== t.id) }))}
                              >
                                <Trash2 className="h-4 w-4" />
                              </Button>
                            </div>
                          </div>
                        ))}
                      </div>
                    )}
                  </CardContent>
                </Card>
              </div>
            </TabsContent>

            {/* Athletics */}
            <TabsContent value="athletics" className="mt-5">
              <div className="grid gap-4 md:grid-cols-3">
                <Card className="rounded-3xl shadow-sm md:col-span-1">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Add</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    <SectionTitle
                      icon={<Dumbbell className="h-4 w-4" />}
                      title="Athletics"
                      subtitle="Practices, games, lifts"
                    />
                    <div className="mt-3">
                      <AddAthleticsDialog onAdd={(evt) => setState((s) => ({ ...s, athletics: [...s.athletics, evt] }))} />
                    </div>
                    <div className="mt-3 text-xs text-slate-500">
                      Use the Day View tab to see athletics + classes + due work together.
                    </div>
                  </CardContent>
                </Card>

                <Card className="rounded-3xl shadow-sm md:col-span-2">
                  <CardHeader className="pb-2">
                    <CardTitle className="text-base">Events</CardTitle>
                  </CardHeader>
                  <CardContent className="pt-0">
                    {state.athletics.length === 0 ? (
                      <EmptyState title="No athletics yet" hint="Add practice or games." />
                    ) : (
                      <Table>
                        <TableHeader>
                          <TableRow>
                            <TableHead>Date</TableHead>
                            <TableHead>Time</TableHead>
                            <TableHead>Event</TableHead>
                            <TableHead>Location</TableHead>
                            <TableHead className="text-right">Actions</TableHead>
                          </TableRow>
                        </TableHeader>
                        <TableBody>
                          {state.athletics
                            .slice()
                            .sort((a, b) => (a.date !== b.date ? a.date.localeCompare(b.date) : a.start.localeCompare(b.start)))
                            .map((e) => (
                              <TableRow key={e.id}>
                                <TableCell className="font-medium">{fmtDate(e.date)}</TableCell>
                                <TableCell>{e.start}–{e.end}</TableCell>
                                <TableCell>{e.title}</TableCell>
                                <TableCell className="text-slate-600">{e.location || ""}</TableCell>
                                <TableCell className="text-right">
                                  <Button
                                    variant="outline"
                                    size="sm"
                                    className="rounded-xl"
                                    onClick={() => setState((s) => ({ ...s, athletics: s.athletics.filter((x) => x.id !== e.id) }))}
                                  >
                                    <Trash2 className="h-4 w-4" />
                                  </Button>
                                </TableCell>
                              </TableRow>
                            ))}
                        </TableBody>
                      </Table>
                    )}
                  </CardContent>
                </Card>
              </div>
            </TabsContent>
          </Tabs>
        </div>

        <footer className="mt-10 text-xs text-slate-500">
          <div className="flex flex-col gap-1 md:flex-row md:items-center md:justify-between">
            <div>Planner data is stored locally in your browser (localStorage).</div>
            <div>Next upgrade: login, sharing, notifications.</div>
          </div>
        </footer>
      </div>
    </div>
  );
}
