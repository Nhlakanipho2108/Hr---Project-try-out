<script setup>
import { computed, reactive, ref, watch } from 'vue'
import employeeInfoData from '../employee_info.json'
import attendanceData from '../attendance.json'
import payrollData from '../payroll_data.json'

const STORAGE_KEY = 'moderntech-hr-poc-state-v2'
const AUTH_USER = 'hr_admin'
const AUTH_PASSWORD = 'MT2026!'

function parseStartDateFromHistory(employmentHistory) {
  const yearMatch = /Joined in\s+(\d{4})/i.exec(employmentHistory ?? '')
  if (!yearMatch) {
    return '2020-01-01'
  }
  return `${yearMatch[1]}-01-01`
}

const importedEmployees = (employeeInfoData.employeeInformation ?? []).map((employee) => ({
  id: employee.employeeId,
  name: employee.name,
  email: employee.contact,
  department: employee.department,
  role: employee.position,
  salary: Number(employee.salary) * 12,
  startDate: parseStartDateFromHistory(employee.employmentHistory),
  history: employee.employmentHistory
}))

const importedAttendance = (attendanceData.attendanceAndLeave ?? []).map((employeeRecord) => ({
  employeeId: employeeRecord.employeeId,
  present: employeeRecord.attendance.filter((entry) => entry.status === 'Present').length,
  remote: employeeRecord.attendance.filter((entry) => entry.status === 'Remote').length,
  absent: employeeRecord.attendance.filter((entry) => entry.status === 'Absent').length,
  leaveDays: employeeRecord.leaveRequests.filter((entry) => entry.status === 'Approved').length
}))

const importedLeaveRequests = []
let leaveRequestId = 1

for (const employeeRecord of attendanceData.attendanceAndLeave ?? []) {
  for (const leaveRequest of employeeRecord.leaveRequests ?? []) {
    importedLeaveRequests.push({
      id: leaveRequestId,
      employeeId: employeeRecord.employeeId,
      startDate: leaveRequest.date,
      endDate: leaveRequest.date,
      reason: leaveRequest.reason,
      status: leaveRequest.status
    })
    leaveRequestId += 1
  }
}

const importedPayrollSource = (payrollData.payrollData ?? []).map((payrollEntry) => ({
  employeeId: payrollEntry.employeeId,
  hoursWorked: payrollEntry.hoursWorked,
  leaveDeductions: payrollEntry.leaveDeductions,
  finalSalary: payrollEntry.finalSalary
}))

const importedPayslips = importedPayrollSource.map((payrollEntry, index) => ({
  id: Date.now() + index,
  month: 'Imported Dataset',
  employeeId: payrollEntry.employeeId,
  baseMonthly: payrollEntry.finalSalary,
  overtimeHours: Math.max(0, payrollEntry.hoursWorked - 160),
  overtimePay: Math.max(0, payrollEntry.hoursWorked - 160) * 180,
  taxDeduction: payrollEntry.finalSalary * 0.18,
  pensionDeduction: payrollEntry.finalSalary * 0.06,
  leaveDeductionAmount: payrollEntry.leaveDeductions * 180,
  netPay: payrollEntry.finalSalary - payrollEntry.finalSalary * 0.18 - payrollEntry.finalSalary * 0.06
}))

const activeTab = ref('dashboard')
const authRole = ref('guest')
const loginError = ref('')
const selectedPayrollEmployeeId = ref(importedEmployees[0]?.id ?? 1)
const selectedPayrollMonth = ref('June 2026')
const employeeLoginError = ref('')
const employeeSessionId = ref(null)

const loginForm = reactive({
  username: '',
  password: ''
})

const employeeLoginForm = reactive({
  employeeId: importedEmployees[0]?.id ?? 1,
  email: ''
})

const employeeForm = reactive({
  name: '',
  email: '',
  department: '',
  role: '',
  salary: '',
  startDate: ''
})

const leaveForm = reactive({
  employeeId: importedEmployees[0]?.id ?? 1,
  startDate: '',
  endDate: '',
  reason: ''
})

const state = reactive({
  employees: importedEmployees,
  leaveRequests: importedLeaveRequests,
  attendance: importedAttendance,
  performanceReviews: [
    {
      id: 1,
      employeeId: importedEmployees[0]?.id ?? 1,
      period: 'Q2 2026',
      rating: 4.7,
      summary: 'Excellent delivery pace and mentoring impact.'
    },
    {
      id: 2,
      employeeId: importedEmployees[1]?.id ?? 2,
      period: 'Q2 2026',
      rating: 4.5,
      summary: 'High defect prevention score in sprint releases.'
    },
    {
      id: 3,
      employeeId: importedEmployees[2]?.id ?? 3,
      period: 'Q2 2026',
      rating: 4.2,
      summary: 'Improved ticket closure by 16 percent.'
    }
  ],
  payrollSource: importedPayrollSource,
  generatedPayslips: importedPayslips
})

const employeeValidationErrors = computed(() => {
  const errors = []

  if (!employeeForm.name.trim()) {
    errors.push('Employee name is required.')
  }
  if (!/^\S+@\S+\.\S+$/.test(employeeForm.email.trim())) {
    errors.push('A valid work email is required.')
  }
  if (!employeeForm.department.trim()) {
    errors.push('Department is required.')
  }
  if (!employeeForm.role.trim()) {
    errors.push('Role is required.')
  }
  if (!Number(employeeForm.salary) || Number(employeeForm.salary) < 120000) {
    errors.push('Salary must be at least R120,000.')
  }
  if (!employeeForm.startDate) {
    errors.push('Start date is required.')
  }

  return errors
})

const leaveValidationErrors = computed(() => {
  const errors = []

  if (!leaveForm.startDate || !leaveForm.endDate) {
    errors.push('Start and end date are required.')
  }
  if (leaveForm.startDate && leaveForm.endDate && leaveForm.startDate > leaveForm.endDate) {
    errors.push('End date must be on or after start date.')
  }
  if (!leaveForm.reason.trim()) {
    errors.push('Please provide a short leave reason.')
  }

  return errors
})

const employeeById = computed(() => {
  return Object.fromEntries(state.employees.map((employee) => [employee.id, employee]))
})

const dashboardStats = computed(() => {
  const pendingRequests = state.leaveRequests.filter((request) => request.status === 'Pending').length
  const annualPayroll = state.employees.reduce((sum, employee) => sum + employee.salary, 0)
  const averageRating =
    state.performanceReviews.reduce((sum, review) => sum + review.rating, 0) /
    state.performanceReviews.length

  return {
    employees: state.employees.length,
    pendingRequests,
    monthlyPayroll: annualPayroll / 12,
    averageRating
  }
})

const attendanceChartData = computed(() => {
  return state.attendance.map((record) => {
    const totalTrackedDays = record.present + record.remote + record.absent + record.leaveDays
    const attendanceScore = ((record.present + record.remote) / totalTrackedDays) * 100

    return {
      ...record,
      name: employeeById.value[record.employeeId]?.name ?? 'Unknown',
      attendanceScore
    }
  })
})

const selectedPayrollEmployee = computed(() => {
  return employeeById.value[selectedPayrollEmployeeId.value]
})

const payrollSourceByEmployee = computed(() => {
  return Object.fromEntries(state.payrollSource.map((entry) => [entry.employeeId, entry]))
})

const employeeSelfProfile = computed(() => {
  if (!employeeSessionId.value) {
    return null
  }
  return employeeById.value[employeeSessionId.value] ?? null
})

const employeeSelfAttendance = computed(() => {
  if (!employeeSessionId.value) {
    return null
  }
  return (
    state.attendance.find((record) => record.employeeId === employeeSessionId.value) ??
    { present: 0, remote: 0, absent: 0, leaveDays: 0 }
  )
})

const employeeSelfLeaveRequests = computed(() => {
  if (!employeeSessionId.value) {
    return []
  }
  return state.leaveRequests.filter((request) => request.employeeId === employeeSessionId.value)
})

const employeeSelfPayroll = computed(() => {
  if (!employeeSessionId.value) {
    return null
  }
  return payrollSourceByEmployee.value[employeeSessionId.value] ?? null
})

const employeeSelfLatestPayslip = computed(() => {
  if (!employeeSessionId.value) {
    return null
  }
  return state.generatedPayslips.find((slip) => slip.employeeId === employeeSessionId.value) ?? null
})

watch(
  state,
  () => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(state))
  },
  { deep: true }
)

bootstrapFromLocalStorage()

function bootstrapFromLocalStorage() {
  const savedState = localStorage.getItem(STORAGE_KEY)
  if (!savedState) {
    return
  }

  try {
    const parsed = JSON.parse(savedState)
    state.employees = parsed.employees ?? state.employees
    state.leaveRequests = parsed.leaveRequests ?? state.leaveRequests
    state.attendance = parsed.attendance ?? state.attendance
    state.performanceReviews = parsed.performanceReviews ?? state.performanceReviews
    state.payrollSource = parsed.payrollSource ?? state.payrollSource
    state.generatedPayslips = parsed.generatedPayslips ?? state.generatedPayslips
  } catch {
    localStorage.removeItem(STORAGE_KEY)
  }
}

function login() {
  if (loginForm.username === AUTH_USER && loginForm.password === AUTH_PASSWORD) {
    authRole.value = 'hr'
    loginError.value = ''
    employeeLoginError.value = ''
    employeeSessionId.value = null
    return
  }

  loginError.value = 'Invalid credentials. Use hr_admin / MT2026! for this demo.'
}

function loginEmployee() {
  const employee = employeeById.value[Number(employeeLoginForm.employeeId)]
  if (!employee) {
    employeeLoginError.value = 'Select a valid employee profile.'
    return
  }

  if (employee.email.toLowerCase() !== employeeLoginForm.email.trim().toLowerCase()) {
    employeeLoginError.value = 'Email does not match selected employee profile.'
    return
  }

  employeeSessionId.value = employee.id
  authRole.value = 'employee'
  employeeLoginError.value = ''
  loginError.value = ''
}

function logout() {
  authRole.value = 'guest'
  employeeSessionId.value = null
  loginForm.username = ''
  loginForm.password = ''
  employeeLoginForm.email = ''
}

function addEmployee() {
  if (employeeValidationErrors.value.length) {
    return
  }

  const newId = Math.max(...state.employees.map((employee) => employee.id)) + 1
  state.employees.push({
    id: newId,
    name: employeeForm.name.trim(),
    email: employeeForm.email.trim(),
    department: employeeForm.department.trim(),
    role: employeeForm.role.trim(),
    salary: Number(employeeForm.salary),
    startDate: employeeForm.startDate,
    history: 'Added through HR portal.'
  })

  state.attendance.push({ employeeId: newId, present: 0, remote: 0, absent: 0, leaveDays: 0 })
  state.payrollSource.push({ employeeId: newId, hoursWorked: 160, leaveDeductions: 0, finalSalary: 0 })

  employeeForm.name = ''
  employeeForm.email = ''
  employeeForm.department = ''
  employeeForm.role = ''
  employeeForm.salary = ''
  employeeForm.startDate = ''
}

function submitLeaveRequest() {
  if (leaveValidationErrors.value.length) {
    return
  }

  const newId = Math.max(0, ...state.leaveRequests.map((request) => request.id)) + 1

  state.leaveRequests.unshift({
    id: newId,
    employeeId: Number(leaveForm.employeeId),
    startDate: leaveForm.startDate,
    endDate: leaveForm.endDate,
    reason: leaveForm.reason.trim(),
    status: 'Pending'
  })

  leaveForm.startDate = ''
  leaveForm.endDate = ''
  leaveForm.reason = ''
}

function updateLeaveStatus(requestId, newStatus) {
  const targetRequest = state.leaveRequests.find((request) => request.id === requestId)
  if (!targetRequest) {
    return
  }

  if (targetRequest.status === newStatus) {
    return
  }

  targetRequest.status = newStatus

  if (newStatus === 'Approved') {
    const attendanceRecord = state.attendance.find(
      (record) => record.employeeId === targetRequest.employeeId
    )
    if (!attendanceRecord) {
      return
    }

    const leaveDays = getDateRangeLength(targetRequest.startDate, targetRequest.endDate)
    attendanceRecord.leaveDays += leaveDays
  }
}

function generatePayslip() {
  const employee = selectedPayrollEmployee.value
  if (!employee) {
    return
  }

  const attendanceRecord = state.attendance.find((record) => record.employeeId === employee.id)
  const payrollBaseline = payrollSourceByEmployee.value[employee.id]

  const overtimeHours = Math.max(
    0,
    payrollBaseline?.hoursWorked ? payrollBaseline.hoursWorked - 160 : (attendanceRecord?.remote ?? 0) * 2
  )

  const baseMonthly = payrollBaseline?.finalSalary ?? employee.salary / 12
  const overtimePay = overtimeHours * 180
  const taxDeduction = (baseMonthly + overtimePay) * 0.18
  const pensionDeduction = baseMonthly * 0.06
  const leaveDeductionAmount = (payrollBaseline?.leaveDeductions ?? 0) * 180
  const netPay = baseMonthly + overtimePay - taxDeduction - pensionDeduction - leaveDeductionAmount

  state.generatedPayslips.unshift({
    id: Date.now(),
    month: selectedPayrollMonth.value,
    employeeId: employee.id,
    baseMonthly,
    overtimeHours,
    overtimePay,
    taxDeduction,
    pensionDeduction,
    leaveDeductionAmount,
    netPay
  })
}

function getDateRangeLength(startDate, endDate) {
  const oneDay = 1000 * 60 * 60 * 24
  const start = new Date(startDate)
  const end = new Date(endDate)
  return Math.floor((end - start) / oneDay) + 1
}

function formatCurrency(amount) {
  return new Intl.NumberFormat('en-ZA', {
    style: 'currency',
    currency: 'ZAR',
    maximumFractionDigits: 0
  }).format(amount)
}

function formatDate(dateValue) {
  return new Date(dateValue).toLocaleDateString('en-ZA', {
    day: '2-digit',
    month: 'short',
    year: 'numeric'
  })
}
</script>

<template>
  <div class="page-shell">
    <div class="gradient-orb orb-one"></div>
    <div class="gradient-orb orb-two"></div>

    <main class="container py-4 py-lg-5">
      <section v-if="authRole === 'guest'" class="row justify-content-center g-3 g-lg-4">
        <div class="col-12 col-md-10 col-lg-6">
          <div class="panel-card p-4 p-lg-5 reveal-in">
            <span class="eyebrow">ModernTech HR Portal</span>
            <h1 class="hero-title mt-2">HR System Proof of Concept</h1>
            <p class="text-muted mb-4">
              Demo login for the mock authentication bonus requirement.
            </p>

            <form @submit.prevent="login" class="row g-3">
              <div class="col-12">
                <label class="form-label">Username</label>
                <input v-model="loginForm.username" class="form-control" placeholder="hr_admin" />
              </div>
              <div class="col-12">
                <label class="form-label">Password</label>
                <input
                  v-model="loginForm.password"
                  type="password"
                  class="form-control"
                  placeholder="MT2026!"
                />
              </div>
              <div v-if="loginError" class="col-12">
                <div class="alert alert-danger mb-0 py-2">{{ loginError }}</div>
              </div>
              <div class="col-12 d-grid">
                <button type="submit" class="btn btn-aurora btn-lg">Access HR Portal</button>
              </div>
            </form>
          </div>
        </div>

        <div class="col-12 col-md-10 col-lg-6">
          <div class="panel-card p-4 p-lg-5 reveal-in">
            <span class="eyebrow">ModernTech Employee Portal</span>
            <h2 class="section-title mt-2">Employee Self-Service Login</h2>
            <p class="text-muted mb-4">
              Employees can view their own profile, salary, start date, leave and payroll summary.
            </p>

            <form @submit.prevent="loginEmployee" class="row g-3">
              <div class="col-12">
                <label class="form-label">Employee</label>
                <select v-model="employeeLoginForm.employeeId" class="form-select">
                  <option v-for="employee in state.employees" :key="employee.id" :value="employee.id">
                    {{ employee.name }}
                  </option>
                </select>
              </div>
              <div class="col-12">
                <label class="form-label">Work Email</label>
                <input
                  v-model="employeeLoginForm.email"
                  type="email"
                  class="form-control"
                  placeholder="employee@moderntech.com"
                />
              </div>
              <div v-if="employeeLoginError" class="col-12">
                <div class="alert alert-danger mb-0 py-2">{{ employeeLoginError }}</div>
              </div>
              <div class="col-12 d-grid">
                <button type="submit" class="btn btn-outline-dark btn-lg">Open My Dashboard</button>
              </div>
            </form>
          </div>
        </div>
      </section>

      <section v-else-if="authRole === 'hr'" class="reveal-in">
        <header class="panel-card p-3 p-lg-4 mb-4 d-flex flex-column flex-lg-row gap-3">
          <div>
            <span class="eyebrow">ModernTech Solutions</span>
            <h1 class="hero-title mb-1">Human Resources Command Center</h1>
            <p class="text-muted mb-0">
              Centralized employee records, leave workflows, payroll, and performance insights.
            </p>
          </div>
          <div class="ms-lg-auto d-flex align-items-start align-items-lg-center">
            <button class="btn btn-outline-dark" @click="logout">Log out</button>
          </div>
        </header>

        <nav class="panel-card mb-4 p-2 overflow-auto">
          <div class="d-flex flex-nowrap gap-2">
            <button
              v-for="tab in ['dashboard', 'employees', 'leave', 'attendance', 'payroll', 'reviews']"
              :key="tab"
              class="btn text-capitalize"
              :class="activeTab === tab ? 'btn-aurora' : 'btn-outline-dark'"
              @click="activeTab = tab"
            >
              {{ tab }}
            </button>
          </div>
        </nav>

        <section v-if="activeTab === 'dashboard'" class="row g-3 g-lg-4">
          <div class="col-6 col-lg-3">
            <article class="metric-card">
              <h2>{{ dashboardStats.employees }}</h2>
              <p>Total employees</p>
            </article>
          </div>
          <div class="col-6 col-lg-3">
            <article class="metric-card">
              <h2>{{ dashboardStats.pendingRequests }}</h2>
              <p>Pending leave requests</p>
            </article>
          </div>
          <div class="col-6 col-lg-3">
            <article class="metric-card">
              <h2>{{ formatCurrency(dashboardStats.monthlyPayroll) }}</h2>
              <p>Estimated monthly payroll</p>
            </article>
          </div>
          <div class="col-6 col-lg-3">
            <article class="metric-card">
              <h2>{{ dashboardStats.averageRating.toFixed(1) }}</h2>
              <p>Average performance rating</p>
            </article>
          </div>

          <div class="col-12">
            <article class="panel-card p-3 p-lg-4">
              <h3 class="section-title">Attendance Compliance (Dummy Chart)</h3>
              <p class="text-muted small mb-4">
                Bonus feature: client-side data visualization based on attendance records.
              </p>

              <div class="d-flex flex-column gap-3">
                <div v-for="item in attendanceChartData" :key="item.employeeId">
                  <div class="d-flex justify-content-between small mb-1">
                    <strong>{{ item.name }}</strong>
                    <span>{{ item.attendanceScore.toFixed(0) }}%</span>
                  </div>
                  <div class="chart-track">
                    <div class="chart-fill" :style="{ width: `${item.attendanceScore}%` }"></div>
                  </div>
                </div>
              </div>
            </article>
          </div>
        </section>

        <section v-if="activeTab === 'employees'" class="row g-3 g-lg-4">
          <div class="col-12 col-xl-4">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">Add Employee</h3>
              <form @submit.prevent="addEmployee" class="row g-3">
                <div class="col-12">
                  <label class="form-label">Full Name</label>
                  <input v-model="employeeForm.name" class="form-control" />
                </div>
                <div class="col-12">
                  <label class="form-label">Work Email</label>
                  <input v-model="employeeForm.email" class="form-control" type="email" />
                </div>
                <div class="col-12">
                  <label class="form-label">Department</label>
                  <input v-model="employeeForm.department" class="form-control" />
                </div>
                <div class="col-12">
                  <label class="form-label">Role</label>
                  <input v-model="employeeForm.role" class="form-control" />
                </div>
                <div class="col-12 col-md-6">
                  <label class="form-label">Annual Salary (ZAR)</label>
                  <input v-model="employeeForm.salary" class="form-control" type="number" />
                </div>
                <div class="col-12 col-md-6">
                  <label class="form-label">Start Date</label>
                  <input v-model="employeeForm.startDate" class="form-control" type="date" />
                </div>

                <div v-if="employeeValidationErrors.length" class="col-12">
                  <div class="alert alert-warning mb-0 py-2">
                    <div v-for="error in employeeValidationErrors" :key="error">{{ error }}</div>
                  </div>
                </div>

                <div class="col-12 d-grid">
                  <button class="btn btn-aurora" type="submit">Save Employee Record</button>
                </div>
              </form>
            </article>
          </div>

          <div class="col-12 col-xl-8">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">Employee Data Repository</h3>
              <div class="table-responsive">
                <table class="table align-middle">
                  <thead>
                    <tr>
                      <th>Name</th>
                      <th>Department</th>
                      <th>Role</th>
                      <th>Salary</th>
                      <th>Employment History</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="employee in state.employees" :key="employee.id">
                      <td>
                        <strong>{{ employee.name }}</strong>
                        <div class="small text-muted">{{ employee.email }}</div>
                      </td>
                      <td>{{ employee.department }}</td>
                      <td>{{ employee.role }}</td>
                      <td>{{ formatCurrency(employee.salary) }}</td>
                      <td>{{ employee.history }}</td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </article>
          </div>
        </section>

        <section v-if="activeTab === 'leave'" class="row g-3 g-lg-4">
          <div class="col-12 col-lg-5">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">Submit Time Off Request</h3>
              <form @submit.prevent="submitLeaveRequest" class="row g-3">
                <div class="col-12">
                  <label class="form-label">Employee</label>
                  <select v-model="leaveForm.employeeId" class="form-select">
                    <option v-for="employee in state.employees" :key="employee.id" :value="employee.id">
                      {{ employee.name }}
                    </option>
                  </select>
                </div>
                <div class="col-12 col-md-6">
                  <label class="form-label">Start Date</label>
                  <input v-model="leaveForm.startDate" class="form-control" type="date" />
                </div>
                <div class="col-12 col-md-6">
                  <label class="form-label">End Date</label>
                  <input v-model="leaveForm.endDate" class="form-control" type="date" />
                </div>
                <div class="col-12">
                  <label class="form-label">Reason</label>
                  <textarea v-model="leaveForm.reason" class="form-control" rows="3"></textarea>
                </div>

                <div v-if="leaveValidationErrors.length" class="col-12">
                  <div class="alert alert-warning mb-0 py-2">
                    <div v-for="error in leaveValidationErrors" :key="error">{{ error }}</div>
                  </div>
                </div>

                <div class="col-12 d-grid">
                  <button class="btn btn-aurora" type="submit">Create Leave Request</button>
                </div>
              </form>
            </article>
          </div>

          <div class="col-12 col-lg-7">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">Approval Queue</h3>
              <div class="d-flex flex-column gap-3">
                <div v-for="request in state.leaveRequests" :key="request.id" class="request-card">
                  <div>
                    <strong>{{ employeeById[request.employeeId]?.name }}</strong>
                    <div class="small text-muted">
                      {{ formatDate(request.startDate) }} to {{ formatDate(request.endDate) }}
                    </div>
                    <div class="small">{{ request.reason }}</div>
                  </div>
                  <div class="text-end">
                    <span
                      class="badge mb-2"
                      :class="
                        request.status === 'Approved'
                          ? 'text-bg-success'
                          : request.status === 'Denied'
                            ? 'text-bg-danger'
                            : 'text-bg-warning'
                      "
                    >
                      {{ request.status }}
                    </span>
                    <div class="d-flex gap-2 justify-content-end">
                      <button class="btn btn-sm btn-success" @click="updateLeaveStatus(request.id, 'Approved')">
                        Approve
                      </button>
                      <button class="btn btn-sm btn-outline-danger" @click="updateLeaveStatus(request.id, 'Denied')">
                        Deny
                      </button>
                    </div>
                  </div>
                </div>
              </div>
            </article>
          </div>
        </section>

        <section v-if="activeTab === 'attendance'" class="row g-3 g-lg-4">
          <div class="col-12">
            <article class="panel-card p-3 p-lg-4">
              <h3 class="section-title">Attendance Tracking</h3>
              <p class="small text-muted">
                Approved leave requests automatically increase leave day totals.
              </p>

              <div class="table-responsive">
                <table class="table align-middle">
                  <thead>
                    <tr>
                      <th>Employee</th>
                      <th>Present</th>
                      <th>Remote</th>
                      <th>Absent</th>
                      <th>Leave Days</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="record in state.attendance" :key="record.employeeId">
                      <td>{{ employeeById[record.employeeId]?.name }}</td>
                      <td>{{ record.present }}</td>
                      <td>{{ record.remote }}</td>
                      <td>{{ record.absent }}</td>
                      <td>{{ record.leaveDays }}</td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </article>
          </div>
        </section>

        <section v-if="activeTab === 'payroll'" class="row g-3 g-lg-4">
          <div class="col-12 col-lg-5">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">Automated Payroll</h3>
              <form @submit.prevent="generatePayslip" class="row g-3">
                <div class="col-12">
                  <label class="form-label">Employee</label>
                  <select v-model="selectedPayrollEmployeeId" class="form-select">
                    <option v-for="employee in state.employees" :key="employee.id" :value="employee.id">
                      {{ employee.name }}
                    </option>
                  </select>
                </div>
                <div class="col-12">
                  <label class="form-label">Pay Period</label>
                  <input v-model="selectedPayrollMonth" class="form-control" />
                </div>
                <div class="col-12 d-grid">
                  <button class="btn btn-aurora" type="submit">Generate Digital Payslip</button>
                </div>
              </form>
            </article>
          </div>

          <div class="col-12 col-lg-7">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">Payroll Source Data</h3>
              <div class="table-responsive mb-3">
                <table class="table align-middle">
                  <thead>
                    <tr>
                      <th>Employee</th>
                      <th>Hours Worked</th>
                      <th>Leave Deductions</th>
                      <th>Final Salary</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="entry in state.payrollSource" :key="entry.employeeId">
                      <td>{{ employeeById[entry.employeeId]?.name }}</td>
                      <td>{{ entry.hoursWorked }}</td>
                      <td>{{ entry.leaveDeductions }}</td>
                      <td>{{ formatCurrency(entry.finalSalary) }}</td>
                    </tr>
                  </tbody>
                </table>
              </div>

              <h3 class="section-title">Generated Payslips</h3>
              <div class="d-flex flex-column gap-2">
                <div v-for="slip in state.generatedPayslips" :key="slip.id" class="payslip-card">
                  <div class="d-flex justify-content-between align-items-start gap-3 flex-wrap">
                    <div>
                      <strong>{{ employeeById[slip.employeeId]?.name }}</strong>
                      <div class="small text-muted">{{ slip.month }}</div>
                      <div class="small">
                        Base: {{ formatCurrency(slip.baseMonthly) }} | Overtime:
                        {{ formatCurrency(slip.overtimePay) }}
                      </div>
                      <div class="small">
                        Tax: {{ formatCurrency(slip.taxDeduction) }} | Pension:
                        {{ formatCurrency(slip.pensionDeduction) }}
                      </div>
                      <div class="small">
                        Leave Deduction: {{ formatCurrency(slip.leaveDeductionAmount ?? 0) }}
                      </div>
                    </div>
                    <div class="text-end">
                      <div class="small text-muted">Net Pay</div>
                      <div class="fw-bold fs-5">{{ formatCurrency(slip.netPay) }}</div>
                    </div>
                  </div>
                </div>
                <p v-if="!state.generatedPayslips.length" class="text-muted mb-0">
                  No payslips generated yet.
                </p>
              </div>
            </article>
          </div>
        </section>

        <section v-if="activeTab === 'reviews'" class="row g-3 g-lg-4">
          <div class="col-12">
            <article class="panel-card p-3 p-lg-4">
              <h3 class="section-title">Performance Review Records</h3>
              <div class="table-responsive">
                <table class="table align-middle">
                  <thead>
                    <tr>
                      <th>Employee</th>
                      <th>Period</th>
                      <th>Rating</th>
                      <th>Summary</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr v-for="review in state.performanceReviews" :key="review.id">
                      <td>{{ employeeById[review.employeeId]?.name }}</td>
                      <td>{{ review.period }}</td>
                      <td>{{ review.rating.toFixed(1) }}</td>
                      <td>{{ review.summary }}</td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </article>
          </div>
        </section>
      </section>

      <section v-else class="reveal-in">
        <header class="panel-card p-3 p-lg-4 mb-4 d-flex flex-column flex-lg-row gap-3">
          <div>
            <span class="eyebrow">Employee Dashboard</span>
            <h1 class="hero-title mb-1">Welcome, {{ employeeSelfProfile?.name }}</h1>
            <p class="text-muted mb-0">
              Your personal HR information from the centralized records.
            </p>
          </div>
          <div class="ms-lg-auto d-flex align-items-start align-items-lg-center">
            <button class="btn btn-outline-dark" @click="logout">Log out</button>
          </div>
        </header>

        <section class="row g-3 g-lg-4">
          <div class="col-12 col-lg-6">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">My Profile</h3>
              <div class="small mb-2"><strong>Name:</strong> {{ employeeSelfProfile?.name }}</div>
              <div class="small mb-2"><strong>Email:</strong> {{ employeeSelfProfile?.email }}</div>
              <div class="small mb-2"><strong>Department:</strong> {{ employeeSelfProfile?.department }}</div>
              <div class="small mb-2"><strong>Role:</strong> {{ employeeSelfProfile?.role }}</div>
              <div class="small mb-2"><strong>Employment Start:</strong> {{ formatDate(employeeSelfProfile?.startDate) }}</div>
              <div class="small mb-0"><strong>Employment History:</strong> {{ employeeSelfProfile?.history }}</div>
            </article>
          </div>

          <div class="col-12 col-lg-6">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">My Payroll Summary</h3>
              <div class="small mb-2"><strong>Annual Salary:</strong> {{ formatCurrency(employeeSelfProfile?.salary ?? 0) }}</div>
              <div class="small mb-2"><strong>Monthly Salary:</strong> {{ formatCurrency((employeeSelfProfile?.salary ?? 0) / 12) }}</div>
              <div class="small mb-2"><strong>Hours Worked:</strong> {{ employeeSelfPayroll?.hoursWorked ?? 0 }}</div>
              <div class="small mb-2"><strong>Leave Deductions:</strong> {{ employeeSelfPayroll?.leaveDeductions ?? 0 }}</div>
              <div class="small mb-2"><strong>Payroll Final Salary:</strong> {{ formatCurrency(employeeSelfPayroll?.finalSalary ?? 0) }}</div>
              <div class="small mb-0">
                <strong>Latest Net Pay:</strong>
                {{ formatCurrency(employeeSelfLatestPayslip?.netPay ?? 0) }}
              </div>
            </article>
          </div>

          <div class="col-12 col-lg-6">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">My Attendance</h3>
              <div class="small mb-2"><strong>Present:</strong> {{ employeeSelfAttendance?.present ?? 0 }}</div>
              <div class="small mb-2"><strong>Remote:</strong> {{ employeeSelfAttendance?.remote ?? 0 }}</div>
              <div class="small mb-2"><strong>Absent:</strong> {{ employeeSelfAttendance?.absent ?? 0 }}</div>
              <div class="small mb-0"><strong>Approved Leave Days:</strong> {{ employeeSelfAttendance?.leaveDays ?? 0 }}</div>
            </article>
          </div>

          <div class="col-12 col-lg-6">
            <article class="panel-card p-3 p-lg-4 h-100">
              <h3 class="section-title">My Leave Requests</h3>
              <div v-if="employeeSelfLeaveRequests.length" class="d-flex flex-column gap-2">
                <div v-for="request in employeeSelfLeaveRequests" :key="request.id" class="request-card">
                  <div class="small"><strong>Date:</strong> {{ formatDate(request.startDate) }}</div>
                  <div class="small"><strong>Reason:</strong> {{ request.reason }}</div>
                  <div class="small"><strong>Status:</strong> {{ request.status }}</div>
                </div>
              </div>
              <p v-else class="small text-muted mb-0">No leave requests found for your profile.</p>
            </article>
          </div>
        </section>
      </section>
    </main>
  </div>
</template>
