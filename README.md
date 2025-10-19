Perfect! Let's build the **Course Enrollment System** from scratch following your complete requirements.

---

## 📦 Step 1: All Mongoose Models (in `/models`)

---

### 1️⃣ `models/Student.js`

```js
const mongoose = require("mongoose");

const studentSchema = new mongoose.Schema({
  name: { type: String, required: true },
  enrolledCourses: [{ type: mongoose.Schema.Types.ObjectId, ref: "Course" }]
});

module.exports = mongoose.model("Student", studentSchema);
```

---

### 2️⃣ `models/Teacher.js`

```js
const mongoose = require("mongoose");

const teacherSchema = new mongoose.Schema({
  name: { type: String, required: true },
  role: {
    type: String,
    enum: ["teacher", "admin"],
    default: "teacher"
  },
  courses: [{ type: mongoose.Schema.Types.ObjectId, ref: "Course" }]
});

module.exports = mongoose.model("Teacher", teacherSchema);
```

---

### 3️⃣ `models/Course.js`

```js
const mongoose = require("mongoose");

const courseSchema = new mongoose.Schema({
  name: { type: String, required: true },
  teacher: { type: mongoose.Schema.Types.ObjectId, ref: "Teacher" },
  students: [{ type: mongoose.Schema.Types.ObjectId, ref: "Student" }]
});

module.exports = mongoose.model("Course", courseSchema);
```

---

✅ **Rules Enforced in Models:**

* Each teacher can store max 5 courses → logic enforced in controller.
* Each course can have max 30 students → logic enforced in controller.
* `role` in `Teacher` distinguishes between regular teacher and superadmin.

---

Next, I will provide all the **route files** (`/routes`) using Express. Shall I continue?





Perfect — we'll place all logic **directly inside route files**, skipping separate controller files.

---

## ✅ Step 3: Complete Route Files with Logic (No separate controllers)

---

### 📄 `routes/studentRoutes.js`

```js
const express = require("express");
const router = express.Router();
const Student = require("../models/Student");
const Course = require("../models/Course");

// Create Student
router.post("/", async (req, res) => {
  try {
    const student = await Student.create({ name: req.body.name });
    res.json(student);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Enroll student in course
router.post("/:studentId/enroll/:courseId", async (req, res) => {
  try {
    const { studentId, courseId } = req.params;
    const course = await Course.findById(courseId).populate("students");
    const student = await Student.findById(studentId);

    if (!course || !student) return res.status(404).json({ message: "Course or Student not found" });

    if (course.students.includes(studentId))
      return res.status(400).json({ message: "Student already enrolled" });

    if (course.students.length >= 30)
      return res.status(400).json({ message: "Course full (max 30 students)" });

    course.students.push(studentId);
    student.enrolledCourses.push(courseId);

    await course.save();
    await student.save();

    res.json({ message: "Enrolled successfully" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Withdraw student from course
router.post("/:studentId/withdraw/:courseId", async (req, res) => {
  try {
    const { studentId, courseId } = req.params;
    const course = await Course.findById(courseId);
    const student = await Student.findById(studentId);

    if (!course || !student) return res.status(404).json({ message: "Not found" });

    course.students = course.students.filter(id => id.toString() !== studentId);
    student.enrolledCourses = student.enrolledCourses.filter(id => id.toString() !== courseId);

    await course.save();
    await student.save();

    res.json({ message: "Withdrawn successfully" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = router;
```

---

### 📄 `routes/teacherRoutes.js`

```js
const express = require("express");
const router = express.Router();
const Teacher = require("../models/Teacher");
const Course = require("../models/Course");
const Student = require("../models/Student");

// Create Teacher
router.post("/", async (req, res) => {
  try {
    const teacher = await Teacher.create({
      name: req.body.name,
      role: req.body.role || "teacher"
    });
    res.json(teacher);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Assign course to teacher (admin only)
router.post("/:adminId/assign/:teacherId/:courseId", async (req, res) => {
  try {
    const { adminId, teacherId, courseId } = req.params;
    const admin = await Teacher.findById(adminId);
    const teacher = await Teacher.findById(teacherId);
    const course = await Course.findById(courseId);

    if (!admin || admin.role !== "admin")
      return res.status(403).json({ message: "Only admin can assign courses" });

    if (!teacher || !course)
      return res.status(404).json({ message: "Teacher or course not found" });

    if (teacher.courses.length >= 5)
      return res.status(400).json({ message: "Teacher already has 5 subjects" });

    teacher.courses.push(course._id);
    course.teacher = teacher._id;

    await teacher.save();
    await course.save();

    res.json({ message: "Course assigned to teacher" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Remove student from course (teacher or admin)
router.post("/:teacherId/remove/:courseId/:studentId", async (req, res) => {
  try {
    const { teacherId, courseId, studentId } = req.params;

    const teacher = await Teacher.findById(teacherId);
    const course = await Course.findById(courseId);
    const student = await Student.findById(studentId);

    if (!teacher || !course || !student)
      return res.status(404).json({ message: "Missing entity" });

    const isOwner = course.teacher && course.teacher.toString() === teacherId;
    const isAdmin = teacher.role === "admin";

    if (!isOwner && !isAdmin)
      return res.status(403).json({ message: "Not authorized to remove student" });

    course.students = course.students.filter(id => id.toString() !== studentId);
    student.enrolledCourses = student.enrolledCourses.filter(id => id.toString() !== courseId);

    await course.save();
    await student.save();

    res.json({ message: "Student removed from course" });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = router;
```

---

### 📄 `routes/courseRoutes.js`

```js
const express = require("express");
const router = express.Router();
const Course = require("../models/Course");

// Create course
router.post("/", async (req, res) => {
  try {
    const course = await Course.create({ name: req.body.name });
    res.json(course);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Get all courses with teacher and student list
router.get("/", async (req, res) => {
  try {
    const courses = await Course.find()
      .populate("teacher", "name role")
      .populate("students", "name");
    res.json(courses);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

module.exports = router;
```

---

### ✅ Summary

* Logic is all handled inline in each route file.
* Admin check, subject and student limits, enroll/withdraw, and removal logic is implemented as per your rules.

---

Next, I’ll give you the final step: **`index.js` setup** to start the server and connect MongoDB. Ready?
