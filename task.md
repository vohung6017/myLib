# PPAP Management Refactoring Plan

## Mục tiêu
Refactor file ppap_management.js (5350 dòng) theo cấu trúc rules.md để code ngắn gọn, dễ maintain hơn.

## Yêu cầu
- ✅ Không tạo file mới
- ✅ Code ngắn gọn, tên hàm ngắn gọn
- ✅ Tái sử dụng code trùng lặp thành utils
- ✅ Comment chỉ ở khối code lớn/phức tạp
- ✅ Không thay đổi chức năng
- ✅ ES6+ syntax
- ✅ Cấu trúc theo rules.md

## Cấu trúc mới theo rules.md

```javascript
// Global variables
let projectList = [];
let currentProject = null;
// ... các biến global khác

// Helper functions
const escapeHtml = (input) => { ... }
const safeCompare = (id1, id2) => { ... }

// Utils
const DateUtils = { ... }
const StatusUtils = { ... }
const DOMUtils = { ... }
const ModalUtils = { ... }

// Fetch data from API
const API = {
    fetchUsers: async () => { ... },
    fetchProjects: async (params) => { ... },
    createProject: async (data) => { ... }
}

// Process data
const processProjectData = (data) => { ... }
const processTaskData = (task) => { ... }

// Load & render
const loadProjects = async () => { ... }
const renderProjectList = () => { ... }
const renderTaskList = (tasks) => { ... }

// Click event handlers
const handleProjectClick = (id) => { ... }
const handleTaskClick = (id) => { ... }
const handleFilterClick = () => { ... }

const loadData = async () => {
    await API.fetchUsers();
    await loadProjects();
    loadSelects();
}

const loadEvents = () => {
    $('#filter_button').on('click', handleFilterClick);
    $('#upload').on('click', handleUploadClick);
    // ... other events
}

document.addEventListener('DOMContentLoaded', () => {
    loadData();
    loadEvents();
});
```

## Chi tiết refactoring

### 1. Global Variables (Dòng 1-80)
**Hiện tại:** Rải rác khắp file  
**Sẽ làm:** Gom lại đầu file
```javascript
// Global variables
let usersCache = [];
let projectList = [];
let currentProject = null;
let selectedTasks = [];
let allTemplateIds = new Set();
let draggedEl = null;
let draggedTask = null;
let currentRACIData = {};
let currentRACIProjectId = null;
let currentTaskDetail = null;
let modalOriginalTitle = null;

const SELECT_CONFIGS = [
    {id: 'ppapFilterStatus', endpoint: '/api/tasks/status'},
    {id: 'ppapFilterPriority', endpoint: '/api/tasks/priorities'},
    {id: 'ppapFilterCustomer', endpoint: '/api/customers'},
    {id: 'ppapFilterModel', endpoint: '/api/models'},
    {id: 'ppapFilterStage', endpoint: '/api/stages'},
    {id: 'ppapFilterDepartment', endpoint: '/api/departments'},
    {id: 'ppapFilterProcess', endpoint: '/api/processes'},
    {id: 'ppapFilterProjectStatus', endpoint: '/api/projects/status'}
]
const SELECT_CACHE = {}
const RACI_PLACEHOLDER = '<span class="raci-cell raci-empty"></span>'
```

### 2. Helper Functions (Dòng 81-200)
**Gom các hàm helper nhỏ:**
```javascript
// Helper functions
const escapeHtml = (input) => {
    if (!input) return ''
    return String(input)
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#39;')
}

const safeCompare = (id1, id2) => {
    if (id1 == null || id2 == null) return id1 === id2
    const str1 = String(id1)
    const str2 = String(id2)
    if (str1 === 'null' || str1 === 'undefined' || str2 === 'null' || str2 === 'undefined') return false
    return str1 === str2
}

const debounce = (fn, wait) => {
    let timeout = null
    return function (...args) {
        clearTimeout(timeout)
        timeout = setTimeout(() => fn.apply(this, args), wait)
    }
}

const findProject = (id) => projectList.find(p => safeCompare(p.id, id)) || null

const getEl = (id) => document.getElementById(id)

const getValue = (el) => el?.value?.trim() || ''

const setValue = (el, val) => el && (el.value = val || '')
```

### 3. Utils Objects (Dòng 201-600)
**Nhóm các utils liên quan:**

#### DateUtils
```javascript
// Utils
const DateUtils = {
    toAPI: (dateStr) => {
        if (!dateStr || dateStr === '-' || String(dateStr).toUpperCase() === 'N/A') return null
        
        const str = String(dateStr).trim()
        
        if (window.moment) {
            const m = window.moment(str, [
                'YYYY/MM/DD', 'YYYY-MM-DD',
                'YYYY/MM/DD HH:mm', 'YYYY-MM-DD HH:mm',
                'YYYY/MM/DD HH:mm:ss', 'YYYY-MM-DD HH:mm:ss'
            ], true)
            
            if (m?.isValid?.()) return m.format('YYYY/MM/DD HH:mm:ss')
        }
        
        const [datePart = '', rawTime = '00:00:00'] = str.split(' ')
        const date = datePart.substring(0, 10).replace(/-/g, '/')
        const [h = '00', m = '00', s = '00'] = rawTime.split(':')
        return `${date} ${h}:${m}:${s}`
    },
    
    toDisplay: (dateStr) => DateUtils.toAPI(dateStr) || '-',
    
    formatForFilter: (value) => {
        if (!value) return ''
        if (typeof value.format === 'function') return value.format('YYYY/MM/DD')
        if (value instanceof Date) {
            const y = value.getFullYear()
            const m = ('0' + (value.getMonth() + 1)).slice(-2)
            const d = ('0' + value.getDate()).slice(-2)
            return `${y}/${m}/${d}`
        }
        return String(value).trim().split(' ')[0].replace(/-/g, '/')
    },
    
    initRangePicker: ($input, fromDate, toDate) => {
        let start = fromDate ? window.moment(fromDate.split(' ')[0], 'YYYY/MM/DD', true) : null
        let end = toDate ? window.moment(toDate.split(' ')[0], 'YYYY/MM/DD', true) : null
        
        $input.daterangepicker({
            startDate: start || window.moment().subtract(3, 'months'),
            endDate: end || window.moment(),
            autoApply: false,
            locale: {format: 'YYYY/MM/DD'}
        })
    },
    
    initSinglePicker: ($input, date) => {
        const start = date ? window.moment(date.split(' ')[0], 'YYYY/MM/DD', true) : null
        $input.daterangepicker({
            singleDatePicker: true,
            startDate: start || new Date(),
            autoApply: false,
            locale: {format: 'YYYY/MM/DD'}
        })
    }
}
```

#### StatusUtils
```javascript
const StatusUtils = {
    normalize: (status) => {
        if (!status) return ''
        return String(status).trim().toUpperCase().replace(/[\s-]/g, '_')
    },
    
    getBadge: (status) => {
        const statusMap = {
            OPEN: 'status-pending',
            PENDING: 'status-gray',
            IN_PROGRESS: 'status-in-progress',
            WAITING_FOR_APPROVAL: 'status-waiting',
            COMPLETED: 'status-completed',
            OVERDUE: 'status-overdue',
            IN_PROCESS: 'status-in-progress',
            OVER_DUE: 'status-overdue',
            CREATED: 'status-na',
            RETURNED: 'status-overdue',
            ON_GOING: 'status-pending',
            CLOSED: 'status-completed'
        }
        return statusMap[StatusUtils.normalize(status)] || 'status-na'
    },
    
    getLabel: (status) => {
        const normalized = StatusUtils.normalize(status)
        return normalized ? normalized.replace(/_/g, ' ') : 'N/A'
    },
    
    isInProgress: (status) => {
        const n = StatusUtils.normalize(status)
        return n === 'IN_PROGRESS' || n === 'IN_PROCESS'
    },
    
    isWaitingApproval: (status) => StatusUtils.normalize(status) === 'WAITING_FOR_APPROVAL'
}
```

#### PriorityUtils
```javascript
const PriorityUtils = {
    getBadge: (priority) => priority ? `priority-${priority.toLowerCase()}` : 'priority-medium',
    getLabel: (priority) => priority ? priority.replace(/_/g, ' ') : 'MEDIUM'
}
```

#### UserUtils
```javascript
const UserUtils = {
    format: (user) => {
        if (!user) return ''
        const id = (user.idCard || '').trim()
        const name = (user.displayName || '').trim()
        if (!id && !name) return ''
        return name ? `${id} - ${name}` : id
    },
    
    getLabelById: (idCard) => {
        if (!idCard) return ''
        const found = usersCache.find(u => String(u.idCard || '') === String(idCard).trim())
        return found ? UserUtils.format(found) : String(idCard).trim()
    },
    
    filter: (keyword) => {
        if (!keyword) return usersCache
        const k = keyword.toLowerCase()
        return usersCache.filter(u => 
            (u.idCard || '').toLowerCase().includes(k) ||
            (u.fullName || '').toLowerCase().includes(k) ||
            (u.displayName || '').toLowerCase().includes(k)
        )
    }
}
```

#### DOMUtils
```javascript
const DOMUtils = {
    show: (id) => DOMUtils.setDisplay(id, 'block'),
    hide: (id) => DOMUtils.setDisplay(id, 'none'),
    setDisplay: (id, val) => {
        const el = getEl(id)
        if (el?.style) el.style.display = val
    },
    setText: (id, text) => {
        const el = getEl(id)
        if (el) el.textContent = text || ''
    },
    getVal: (el) => {
        if (!el) return ''
        const tag = el.tagName?.toLowerCase()
        if (['input', 'select', 'textarea'].includes(tag)) return el.value || ''
        return el.textContent || ''
    },
    setVal: (el, val) => {
        if (!el) return
        const tag = el.tagName?.toLowerCase()
        if (['input', 'select', 'textarea'].includes(tag)) el.value = val || ''
        else el.textContent = val || ''
    }
}
```

#### ModalUtils
```javascript
const ModalUtils = {
    open: (modalId) => {
        const el = typeof modalId === 'string' ? getEl(modalId) : modalId
        if (!el) return null
        try {
            const modal = new bootstrap.Modal(el)
            modal.show()
            return modal
        } catch (e) {
            el.classList.add('active')
            return null
        }
    },
    
    close: (modalEl) => {
        if (!modalEl) return
        try {
            const inst = bootstrap.Modal.getInstance(modalEl)
            if (inst?.hide) inst.hide()
            else new bootstrap.Modal(modalEl).hide()
        } catch (e) {
            modalEl.classList.remove('show', 'active')
        }
        setTimeout(() => ModalUtils.cleanupBackdrops(), 150)
    },
    
    cleanupBackdrops: () => {
        const openModals = document.querySelectorAll('.modal.show')
        const backdrops = document.querySelectorAll('.modal-backdrop')
        
        if (backdrops.length > openModals.length) {
            for (let i = openModals.length; i < backdrops.length; i++) {
                backdrops[i]?.parentNode?.removeChild(backdrops[i])
            }
        }
        
        if (openModals.length === 0) {
            document.body.classList.remove('modal-open')
            document.body.style.paddingRight = ''
            document.body.style.overflow = ''
        }
    },
    
    cleanup: () => {
        const openModals = document.querySelectorAll('.modal.show')
        if (openModals.length === 0) {
            document.querySelectorAll('.modal-backdrop').forEach(b => b?.parentNode?.removeChild(b))
            document.body.classList.remove('modal-open')
            document.body.style.paddingRight = ''
            document.body.style.overflow = ''
        }
    }
}
```

#### AlertUtils
```javascript
const AlertUtils = {
    success: (title, text) => Swal.fire({title, text, icon: 'success', customClass: 'swal-success'}),
    error: (title, text) => Swal.fire({title, text, icon: 'error', customClass: 'swal-error'}),
    warning: (title, text) => Swal.fire({title, text, icon: 'warning', customClass: 'swal-warning'}),
    confirm: async (title, text) => {
        const result = await Swal.fire({
            title, text, icon: 'question',
            showCancelButton: true,
            confirmButtonText: 'Confirm',
            cancelButtonText: 'Cancel'
        })
        return result.isConfirmed
    }
}
```

#### ValidationUtils
```javascript
const ValidationUtils = {
    required: (val, name) => {
        const v = val ? String(val).trim() : ''
        if (!v) throw new Error(`${name} is required`)
        return v
    },
    maxLength: (val, max, name) => {
        if (String(val).length > max) throw new Error(`${name} must be less than ${max} characters`)
        return val
    }
}
```

#### SelectUtils
```javascript
const SelectUtils = {
    populate: (selectId, items) => {
        const select = getEl(selectId)
        if (!select) return
        
        let html = '<option value="">--Select--</option>'
        items.forEach(item => {
            if (typeof item === 'string' || typeof item === 'number') {
                html += `<option value="${item}">${item}</option>`
            } else if (item?.id && item?.name) {
                html += `<option value="${item.id}">${item.name}</option>`
            }
        })
        select.innerHTML = html
    },
    
    getNameById: (cacheKey, id) => {
        if (!id) return ''
        const list = SELECT_CACHE[cacheKey] || []
        const found = list.find(item => String(item?.id || '') === String(id).trim())
        return found?.name || found?.label || ''
    },
    
    initDRI: (selectEl, parent) => {
        if (!selectEl || !window.jQuery || !$.fn.select2) return
        
        let target = selectEl
        if (selectEl.tagName === 'INPUT') {
            const val = selectEl.value || ''
            const newSelect = document.createElement('select')
            newSelect.id = selectEl.id
            newSelect.className = selectEl.className
            newSelect.name = selectEl.name || selectEl.id
            Object.assign(newSelect.dataset, selectEl.dataset)
            selectEl.parentNode.replaceChild(newSelect, selectEl)
            target = newSelect
            SelectUtils.populateDRI(target, val)
        } else {
            SelectUtils.populateDRI(target, selectEl.value)
        }
        
        const $select = $(target)
        if ($select.data('select2')) {
            try { $select.select2('destroy') } catch (e) {}
        }
        
        $select.select2({
            placeholder: 'Search user...',
            allowClear: true,
            width: '100%',
            dropdownParent: parent ? $(parent) : $select.parent()
        })
    },
    
    populateDRI: (selectEl, selectedValue) => {
        if (!selectEl) return
        selectEl.innerHTML = '<option value="">-- Select DRI --</option>'
        usersCache.forEach(user => {
            const id = user.idCard || ''
            const option = document.createElement('option')
            option.value = id
            option.textContent = UserUtils.format(user)
            if (selectedValue && id === selectedValue) option.selected = true
            selectEl.appendChild(option)
        })
    }
}
```

### 4. API Functions (Dòng 601-900)
**Nhóm tất cả API calls:**
```javascript
// Fetch data from API
const API = {
    base: '/sample-system',
    
    fetchUsers: async () => {
        const res = await fetch(`${API.base}/api/users`)
        if (!res.ok) throw new Error(`Failed to fetch users: ${res.status}`)
        const json = await res.json()
        return json.result || []
    },
    
    fetchProjects: async (params) => {
        const query = params ? '?' + new URLSearchParams(params) : ''
        const res = await fetch(`${API.base}/api/projects${query}`)
        if (!res.ok) throw new Error(`Failed to fetch projects: ${res.status}`)
        const json = await res.json()
        return Array.isArray(json.data) ? json.data : Array.isArray(json) ? json : []
    },
    
    createProject: async (data) => {
        const res = await fetch(`${API.base}/api/projects/create`, {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify(data)
        })
        if (!res.ok) throw new Error(`Failed to create project: ${res.status}`)
        return await res.json()
    },
    
    updateProject: async (id, taskIds) => {
        const res = await fetch(`${API.base}/api/projects/${id}/update`, {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify(taskIds)
        })
        if (!res.ok) throw new Error(`Failed to update project: ${res.status}`)
        return await res.json()
    },
    
    deleteProject: async (id) => {
        const res = await fetch(`${API.base}/api/projects/${id}`, {method: 'DELETE'})
        if (!res.ok) throw new Error(`Failed to delete project: ${res.status}`)
        return await res.json()
    },
    
    approveProject: async (id) => {
        const res = await fetch(`${API.base}/api/projects/${id}/approve`, {method: 'POST'})
        if (!res.ok) throw new Error(`Failed to approve project: ${res.status}`)
        return await res.json()
    },
    
    rejectProject: async (id) => {
        const res = await fetch(`${API.base}/api/projects/${id}/reject`, {method: 'POST'})
        if (!res.ok) throw new Error(`Failed to reject project: ${res.status}`)
        return await res.json()
    },
    
    fetchTask: async (id) => {
        const res = await fetch(`${API.base}/api/tasks/${id}`)
        if (!res.ok) throw new Error(`Failed to fetch task: ${res.status}`)
        return await res.json()
    },
    
    createTask: async (data) => {
        const res = await fetch(`${API.base}/api/tasks/create`, {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify(data)
        })
        if (!res.ok) throw new Error(`Failed to create task: ${res.status}`)
        return await res.json()
    },
    
    updateTask: async (id, data) => {
        const res = await fetch(`${API.base}/api/tasks/${id}`, {
            method: 'PUT',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify(data)
        })
        if (!res.ok) throw new Error(`Failed to update task: ${res.status}`)
        return await res.json()
    },
    
    removeTask: async (projectId, taskId) => {
        const res = await fetch(`${API.base}/api/projects/${projectId}/tasks/${taskId}`, {method: 'DELETE'})
        if (!res.ok) throw new Error(`Failed to remove task: ${res.status}`)
        return await res.json()
    },
    
    uploadFiles: async (taskId, formData) => {
        const res = await fetch(`${API.base}/api/tasks/upload-files`, {
            method: 'POST',
            body: formData
        })
        if (!res.ok) throw new Error(`Failed to upload files: ${res.status}`)
        return await res.json()
    },
    
    fetchComments: async (taskId) => {
        const res = await fetch(`${API.base}/api/tasks/${taskId}/comments`)
        if (!res.ok) throw new Error(`Failed to fetch comments: ${res.status}`)
        return await res.json()
    },
    
    postComment: async (taskId, text) => {
        const params = new URLSearchParams()
        params.append('comment', text)
        const res = await fetch(`${API.base}/api/tasks/${taskId}/comment`, {
            method: 'POST',
            headers: {'Content-Type': 'application/x-www-form-urlencoded'},
            body: params.toString()
        })
        if (!res.ok) throw new Error(`Failed to post comment: ${res.status}`)
        return await res.json()
    },
    
    fetchOptions: async (endpoint) => {
        const res = await fetch(`${API.base}${endpoint}`)
        if (!res.ok) throw new Error(`Failed to fetch options: ${res.status}`)
        const json = await res.json()
        return json.data || []
    },
    
    fetchRACIData: async (projectId) => {
        const res = await fetch(`${API.base}/api/raci/${projectId}`)
        if (!res.ok) throw new Error(`Failed to fetch RACI: ${res.status}`)
        return await res.json()
    },
    
    saveRACIData: async (projectId, data) => {
        const res = await fetch(`${API.base}/api/raci/${projectId}`, {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify(data)
        })
        if (!res.ok) throw new Error(`Failed to save RACI: ${res.status}`)
        return await res.json()
    }
}
```

### 5. Data Processing (Dòng 901-1100)
```javascript
// Process data
const mapCustomer = (cust) => {
    if (!cust) return ''
    const s = String(cust).trim().toLowerCase()
    if (s === '1' || s === 'apollo') return 1
    if (s === '2' || s === 'rhea') return 2
    if (s === '3' || s === 'kronos') return 3
    const num = Number(s)
    return !isNaN(num) && num > 0 ? num : cust
}

const buildFilterParams = () => {
    const params = {}
    const filters = {
        projectName: 'ppapFilterProject',
        customerId: 'ppapFilterCustomer',
        model: 'filter-model',
        status: 'ppapFilterProjectStatus',
        priority: 'ppapFilterPriority',
        stageId: 'ppapFilterStage',
        departmentId: 'ppapFilterDepartment',
        processId: 'ppapFilterProcess',
        createdBy: 'filter-created-by'
    }
    
    Object.entries(filters).forEach(([key, id]) => {
        const val = getValue(getEl(id))
        if (val) params[key] = key === 'customerId' ? mapCustomer(val) : val
    })
    
    const dateRange = getValue(getEl('filter-created-date'))
    if (dateRange) {
        const [from, to] = dateRange.split('-').map(s => s.trim())
        if (from) params.createdFrom = DateUtils.toAPI(from)
        if (to) params.createdTo = DateUtils.toAPI(to)
    }
    
    return params
}

const parseTaskUpdates = (content) => {
    const fields = Array.from(content.matchAll(/\[(\w+):/g)).map(m => m[1])
    const labels = {
        dri: 'DRI', dueDate: 'Deadline', status: 'Status',
        priority: 'Priority', stageId: 'Stage', processId: 'Process'
    }
    return fields.map(f => labels[f] || f).join(', ') || 'Task updated'
}
```

### 6. Load & Render Functions (Dòng 1101-2200)
```javascript
// Load & render data
const loadProjects = async () => {
    try {
        projectList = await API.fetchProjects()
        renderProjects()
    } catch (e) {
        console.error('Load projects error:', e)
        AlertUtils.error('Error', 'Failed to load projects')
    }
}

const renderProjects = () => {
    const waiting = projectList.filter(p => p.status === 'WAITING_FOR_APPROVAL')
    const others = projectList.filter(p => p.status !== 'WAITING_FOR_APPROVAL')
    
    renderProjectTable('waitingApprovalBody', waiting)
    renderProjectTable('otherProjectsBody', others)
    initDragDrop()
}

const renderProjectTable = (bodyId, projects) => {
    const tbody = getEl(bodyId)
    if (!tbody) return
    
    if (!projects?.length) {
        tbody.innerHTML = '<tr><td colspan="8" class="text-center">No data</td></tr>'
        return
    }
    
    tbody.innerHTML = projects.map((p, i) => `
        <tr draggable="true" data-project-id="${p.id}">
            <td>${i + 1}</td>
            <td>${escapeHtml(p.customer)}</td>
            <td><a href="#" onclick="showProjectModal('${p.id}')">${escapeHtml(p.name)}</a></td>
            <td>${escapeHtml(p.model || '-')}</td>
            <td>${p.taskCount || 0}</td>
            <td><span class="${StatusUtils.getBadge(p.status)}">${StatusUtils.getLabel(p.status)}</span></td>
            <td>${DateUtils.toDisplay(p.createdAt)}</td>
            <td>
                <button onclick="handleProjectEdit('${p.id}')">Edit</button>
                <button onclick="handleProjectDelete('${p.id}')">Delete</button>
            </td>
        </tr>
    `).join('')
}

const loadUsers = async () => {
    if (usersCache.length === 0) {
        usersCache = await API.fetchUsers()
    }
    initDRISelects()
}

const initDRISelects = () => {
    const filters = ['filter-created-by', 'custom-dri', 'dri']
    filters.forEach(id => {
        const el = getEl(id)
        if (el) SelectUtils.initDRI(el, el.closest('.modal'))
    })
}

const loadSelects = async () => {
    const results = await Promise.all(SELECT_CONFIGS.map(cfg => API.fetchOptions(cfg.endpoint)))
    
    SELECT_CONFIGS.forEach((cfg, i) => {
        SelectUtils.populate(cfg.id, results[i])
        SELECT_CACHE[cfg.endpoint] = results[i]
    })
}

const loadComments = async (taskId) => {
    const container = getEl('comment-container')
    if (!container) return
    
    try {
        container.innerHTML = '<div class="loading">Loading...</div>'
        const json = await API.fetchComments(taskId)
        const items = json.data || []
        
        if (!items.length) {
            container.innerHTML = '<div class="empty">No comments</div>'
            return
        }
        
        container.innerHTML = items.map(it => {
            const author = it.createdBy || it.author || '-'
            const date = it.createdAt || it.date || '-'
            const content = it.content || ''
            const type = it.type || 'COMMENT'
            
            if (type === 'LOG') {
                return `<div class="log-item">${escapeHtml(author)} ${escapeHtml(content)} - ${escapeHtml(date)}</div>`
            }
            return `<div class="comment-item"><strong>${escapeHtml(author)}</strong> - ${escapeHtml(date)}<br>${escapeHtml(content)}</div>`
        }).join('')
    } catch (e) {
        console.error('Load comments error:', e)
        container.innerHTML = '<div class="error">Failed to load comments</div>'
    }
}
```

### 7. Click Event Handlers (Dòng 2201-4000)
```javascript
// Click event handlers
const handleProjectFilter = async () => {
    try {
        const params = buildFilterParams()
        projectList = await API.fetchProjects(params)
        renderProjects()
    } catch (e) {
        AlertUtils.error('Error', 'Filter failed')
    }
}

const handleProjectClear = () => {
    ['ppapFilterProject', 'ppapFilterCustomer', 'filter-model', 'ppapFilterProjectStatus',
     'ppapFilterPriority', 'ppapFilterStage', 'ppapFilterDepartment', 'ppapFilterProcess',
     'filter-created-by', 'filter-created-date'].forEach(id => setValue(getEl(id), ''))
    loadProjects()
}

const handleProjectCreate = () => ModalUtils.open('createProjectModal')

const handleProjectDelete = async (id) => {
    const confirmed = await AlertUtils.confirm('Delete', 'Are you sure?')
    if (!confirmed) return
    
    try {
        await API.deleteProject(id)
        AlertUtils.success('Success', 'Project deleted')
        loadProjects()
    } catch (e) {
        AlertUtils.error('Error', 'Delete failed')
    }
}

const handleProjectApprove = async (id) => {
    try {
        await API.approveProject(id)
        AlertUtils.success('Success', 'Project approved')
        loadProjects()
    } catch (e) {
        AlertUtils.error('Error', 'Approve failed')
    }
}

const handleProjectReject = async (id) => {
    try {
        await API.rejectProject(id)
        AlertUtils.success('Success', 'Project rejected')
        loadProjects()
    } catch (e) {
        AlertUtils.error('Error', 'Reject failed')
    }
}

const handleTaskUpload = async () => {
    const modal = getEl('taskDetailModal')
    const taskId = modal?.dataset?.taskId
    if (!taskId) return
    
    const input = document.createElement('input')
    input.type = 'file'
    input.multiple = true
    input.onchange = async () => {
        if (!input.files?.length) return
        
        const formData = new FormData()
        formData.append('id', taskId)
        Array.from(input.files).forEach(f => formData.append('files', f))
        
        try {
            await API.uploadFiles(taskId, formData)
            AlertUtils.success('Success', 'Files uploaded')
        } catch (e) {
            AlertUtils.error('Error', 'Upload failed')
        }
    }
    input.click()
}

const handleTaskComment = async () => {
    const modal = getEl('taskDetailModal')
    const taskId = modal?.dataset?.taskId
    const input = getEl('input-comment')
    const text = getValue(input)
    
    if (!taskId || !text) {
        AlertUtils.warning('Warning', 'Please enter comment')
        return
    }
    
    try {
        await API.postComment(taskId, text)
        AlertUtils.success('Success', 'Comment posted')
        setValue(input, '')
        loadComments(taskId)
    } catch (e) {
        AlertUtils.error('Error', 'Comment failed')
    }
}

const handleFilterEnter = (e) => {
    if (e.key === 'Enter') {
        e.preventDefault()
        handleProjectFilter()
    }
}

const handleSearchTask = debounce((query) => {
    const tbody = getEl('projectTasksTable')?.querySelector('tbody')
    if (!tbody) return
    
    const q = query.trim().toLowerCase()
    const rows = tbody.querySelectorAll('tr')
    
    rows.forEach(row => {
        if (!q) {
            row.style.display = ''
            return
        }
        const text = row.textContent.toLowerCase()
        row.style.display = text.includes(q) ? '' : 'none'
    })
}, 200)
```

### 8. Main Init (Dòng 4001-4100)
```javascript
// Main initialization
const loadData = async () => {
    await loadUsers()
    await loadSelects()
    await loadProjects()
    
    const taskId = new URLSearchParams(location.search).get('taskId')
    const projectId = new URLSearchParams(location.search).get('projectId')
    
    if (taskId) loadTaskDetail(taskId)
    if (projectId) loadProjectTasks(projectId)
}

const loadEvents = () => {
    // File upload
    const upload = getEl('upload')
    if (upload) upload.addEventListener('click', handleTaskUpload)
    
    // Comment
    const comment = getEl('comment')
    if (comment) comment.addEventListener('click', handleTaskComment)
    
    // Filter
    const filterBtn = getEl('filter_button')
    if (filterBtn) filterBtn.addEventListener('click', handleProjectFilter)
    
    const clearBtn = getEl('clear_filter_button')
    if (clearBtn) clearBtn.addEventListener('click', handleProjectClear)
    
    // Enter keys
    ['ppapFilterProject', 'filter-model'].forEach(id => {
        const el = getEl(id)
        if (el) el.addEventListener('keydown', handleFilterEnter)
    })
    
    // Search
    const search = getEl('search-task')
    if (search) search.addEventListener('input', (e) => handleSearchTask(e.target.value))
    
    // Date picker
    if (window.jQuery?.fn?.daterangepicker) {
        const dateInput = getEl('filter-created-date')
        if (dateInput) DateUtils.initRangePicker($(dateInput), null, null)
    }
    
    // Modal cleanup
    const taskModal = getEl('taskDetailModal')
    if (taskModal) {
        taskModal.addEventListener('hidden.bs.modal', () => {
            const url = new URL(location.href)
            url.searchParams.delete('taskId')
            history.pushState({}, '', url)
        })
    }
    
    const projectModal = getEl('projectTasksModal')
    if (projectModal) {
        projectModal.addEventListener('hidden.bs.modal', () => {
            const url = new URL(location.href)
            url.searchParams.delete('projectId')
            history.pushState({}, '', url)
        })
    }
    
    // Global modal cleanup
    document.addEventListener('hidden.bs.modal', () => {
        setTimeout(() => ModalUtils.cleanup(), 10)
    })
}

document.addEventListener('DOMContentLoaded', () => {
    loadData()
    loadEvents()
})
```

## Ước tính kết quả

| Phần | Trước | Sau | Giảm |
|------|-------|-----|------|
| Global variables | ~100 dòng rải rác | ~50 dòng | -50% |
| Utils | ~500 dòng | ~400 dòng | -20% |
| API calls | ~800 dòng | ~500 dòng | -37% |
| Load/Render | ~1000 dòng | ~700 dòng | -30% |
| Event handlers | ~2500 dòng | ~1800 dòng | -28% |
| Init | ~450 dòng | ~100 dòng | -77% |
| **Tổng** | **~5350 dòng** | **~3550 dòng** | **-33%** |

## Lợi ích

1. ✅ **Dễ đọc:** Cấu trúc rõ ràng, tìm code nhanh
2. ✅ **Dễ maintain:** Mỗi function 1 chức năng
3. ✅ **Tái sử dụng:** Utils dùng nhiều nơi
4. ✅ **Ít lỗi:** Code ngắn gọn, ít phức tạp
5. ✅ **ES6+:** Modern syntax
6. ✅ **Tuân thủ rules.md:** Đúng chuẩn

## Thứ tự thực hiện

1. ⬜ Gom global variables lên đầu
2. ⬜ Tạo helper functions
3. ⬜ Tạo Utils objects (Date, Status, Priority, User, DOM, Modal, Alert, Validation, Select)
4. ⬜ Tạo API object
5. ⬜ Tạo data processing functions
6. ⬜ Refactor load & render functions
7. ⬜ Refactor event handlers
8. ⬜ Refactor main init (loadData, loadEvents)
9. ⬜ Test toàn bộ chức năng
10. ⬜ Clean up comments
