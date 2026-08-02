---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "create AI-powered user management dashboard"
  - "implement task tracking with Kanban board"
  - "add AI analytics to user management"
  - "build admin dashboard with user analytics"
  - "integrate AI ticket classification system"
  - "create user management with burnout detection"
  - "implement JWT authentication for enterprise app"
---

# Enterprise User Management AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with AI-powered insights. The system provides role-based access control, Kanban-style task tracking, support ticket management, and ML-driven analytics including risk detection, anomaly detection, burnout analysis, and predictive project insights.

**Architecture:**
- Frontend: React.js
- Backend: Node.js/Express with REST APIs
- ML Service: FastAPI with scikit-learn and River
- Database: MongoDB
- Authentication: JWT

## Installation

### Full Stack Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt
```

### Environment Configuration

**Backend (.env):**
```bash
# backend/.env
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=24h
NODE_ENV=development
```

**ML Service (.env):**
```bash
# ml-service/.env
MODEL_PATH=./models
LOG_LEVEL=info
API_PORT=8000
```

**Frontend (.env):**
```bash
# frontend/.env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the Application

### Start Backend Server
```bash
cd backend
npm start
# Runs on http://localhost:5000
```

### Start ML Service
```bash
cd ml-service
uvicorn main:app --reload
# Runs on http://localhost:8000
```

### Start Frontend
```bash
cd frontend
npm start
# Runs on http://localhost:3000
```

## Backend API Usage

### Authentication

**User Registration:**
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return await response.json();
};
```

**User Login:**
```javascript
// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

**Get All Users:**
```javascript
// GET /api/users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};
```

**Create User:**
```javascript
// POST /api/users
const createUser = async (userData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(userData)
  });
  return await response.json();
};
```

**Update User:**
```javascript
// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};
```

**Delete User:**
```javascript
// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

### Task Management

**Get User Tasks:**
```javascript
// GET /api/tasks
const getUserTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

**Create Task:**
```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: taskData.status || 'todo' // 'todo', 'in-progress', 'done'
    })
  });
  return await response.json();
};
```

**Update Task Status:**
```javascript
// PATCH /api/tasks/:id
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};
```

### Support Tickets

**Create Ticket:**
```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};
```

**Get User Tickets:**
```javascript
// GET /api/tickets/user
const getUserTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets/user', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

## ML Service API Usage

### AI Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Issue with login"
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.89 }
  return result;
};
```

### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ userId })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId,
      tasksCompleted: 45,
      hoursWorked: 52,
      weekendWork: true,
      overtimeHours: 12
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.82, recommendations: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      loginTime: activityData.loginTime,
      location: activityData.location,
      device: activityData.device,
      actionsPerHour: activityData.actionsPerHour
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.91, reasons: [...] }
  return result;
};
```

### Project Delay Prediction

```javascript
// POST /api/ml/predict-project-delay
const predictProjectDelay = async (projectData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:8000/api/ml/predict-project-delay', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      projectId: projectData.projectId,
      tasksCompleted: projectData.tasksCompleted,
      totalTasks: projectData.totalTasks,
      daysRemaining: projectData.daysRemaining,
      teamSize: projectData.teamSize
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.68, estimatedDelay: 5, recommendations: [...] }
  return result;
};
```

## React Frontend Patterns

### Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const navigate = useNavigate();

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      // Verify token and get user data
      fetchUserData(token);
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUserData = async (token) => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/me', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (credentials) => {
    const data = await loginUser(credentials);
    setUser(data.user);
    navigate('/dashboard');
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
    navigate('/login');
  };

  return { user, loading, login, logout };
};
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    const allTasks = await getUserTasks();
    const categorized = {
      todo: allTasks.filter(t => t.status === 'todo'),
      inProgress: allTasks.filter(t => t.status === 'in-progress'),
      done: allTasks.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const moveTask = async (taskId, newStatus) => {
    await updateTaskStatus(taskId, newStatus);
    loadTasks();
  };

  return (
    <div className="kanban-board">
      <Column title="To Do" tasks={tasks.todo} status="todo" onMove={moveTask} />
      <Column title="In Progress" tasks={tasks.inProgress} status="in-progress" onMove={moveTask} />
      <Column title="Done" tasks={tasks.done} status="done" onMove={moveTask} />
    </div>
  );
};

const Column = ({ title, tasks, status, onMove }) => {
  return (
    <div className="kanban-column">
      <h3>{title}</h3>
      {tasks.map(task => (
        <TaskCard key={task._id} task={task} onMove={onMove} />
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### Admin Dashboard Component

```javascript
// components/AdminDashboard.jsx
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    const usersData = await getAllUsers();
    setUsers(usersData);
    
    // Load analytics
    const analyticsData = await fetchAnalytics();
    setAnalytics(analyticsData);
  };

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch('http://localhost:5000/api/analytics/overview', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    return await response.json();
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      <div className="analytics-cards">
        <StatCard title="Total Users" value={analytics?.totalUsers || 0} />
        <StatCard title="Active Tasks" value={analytics?.activeTasks || 0} />
        <StatCard title="Open Tickets" value={analytics?.openTickets || 0} />
        <StatCard title="Risk Alerts" value={analytics?.riskAlerts || 0} />
      </div>
      <UserManagementTable users={users} onRefresh={loadDashboardData} />
    </div>
  );
};

export default AdminDashboard;
```

### Time Tracking Component

```javascript
// components/TimeTracker.jsx
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  const saveTimeLog = async () => {
    const token = localStorage.getItem('token');
    await fetch('http://localhost:5000/api/tasks/time-log', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        taskId,
        duration: seconds
      })
    });
  };

  const handleStop = () => {
    setIsRunning(false);
    saveTimeLog();
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      {seconds > 0 && <button onClick={handleStop}>Stop & Save</button>}
    </div>
  );
};

export default TimeTracker;
```

## Configuration

### MongoDB Schema Examples

**User Schema:**
```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

module.exports = mongoose.model('User', userSchema);
```

**Task Schema:**
```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['todo', 'in-progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high'], 
    default: 'medium' 
  },
  dueDate: Date,
  timeSpent: { type: Number, default: 0 }, // in seconds
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

**Ticket Schema:**
```javascript
// models/Ticket.js
const mongoose = require('mongoose');

const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { 
    type: String, 
    enum: ['open', 'in-progress', 'resolved', 'closed'], 
    default: 'open' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'urgent'], 
    default: 'medium' 
  },
  category: String,
  aiClassification: {
    category: String,
    confidence: Number
  },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = mongoose.model('Ticket', ticketSchema);
```

### JWT Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const authMiddleware = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    
    if (!token) {
      return res.status(401).json({ error: 'Authentication required' });
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.userId);

    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    req.user = user;
    req.token = token;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminMiddleware = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminMiddleware };
```

## Troubleshooting

### JWT Token Expired
```javascript
// Handle token expiration gracefully
const apiCall = async (url, options) => {
  const response = await fetch(url, options);
  
  if (response.status === 401) {
    // Token expired, redirect to login
    localStorage.removeItem('token');
    window.location.href = '/login';
    throw new Error('Session expired');
  }
  
  return response.json();
};
```

### MongoDB Connection Issues
```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Service Not Responding
```bash
# Check ML service logs
cd ml-service
uvicorn main:app --reload --log-level debug

# Verify Python dependencies
pip list | grep -E "fastapi|scikit-learn|river"

# Test ML endpoint
curl -X POST http://localhost:8000/api/ml/health
```

### CORS Issues
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Task Status Not Updating
```javascript
// Ensure proper state management
const updateTaskStatus = async (taskId, newStatus) => {
  try {
    const token = localStorage.getItem('token');
    const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus, updatedAt: new Date() })
    });
    
    if (!response.ok) {
      throw new Error('Failed to update task');
    }
    
    return await response.json();
  } catch (error) {
    console.error('Task update error:', error);
    throw error;
  }
};
```

### AI Model Performance Issues
```python
# ml-service/main.py - Add caching for frequent predictions
from functools import lru_cache
from fastapi import FastAPI

app = FastAPI()

@lru_cache(maxsize=100)
def get_cached_prediction(user_id: str, data_hash: str):
    # Cache predictions based on user and data
    return cached_result
```
