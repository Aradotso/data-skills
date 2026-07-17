---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket routing, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system with AI"
  - "implement user management with AI analytics"
  - "configure JWT authentication for user system"
  - "integrate AI ticket classification and routing"
  - "build admin dashboard with task management"
  - "add burnout detection and risk prediction"
  - "create Kanban board with time tracking"
  - "deploy user management system with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. It provides:

- **User Management**: Secure authentication, role-based access control, user CRUD operations
- **Task Management**: Kanban boards, time tracking, task assignment and monitoring
- **Support System**: Ticket creation, AI-based classification and routing
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, predictive project insights
- **Dashboards**: Separate admin and user interfaces with real-time updates

Built with React frontend, Node.js backend, MongoDB database, and FastAPI ML service.

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+ and pip
- MongoDB (local or cloud instance)

### Clone and Setup

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install
```

Create `.env` file:

```bash
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file:

```bash
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Architecture

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   React     │ ─────▶  │   Node.js    │ ─────▶  │   MongoDB   │
│  Frontend   │         │   Backend    │         │  Database   │
└─────────────┘         └──────────────┘         └─────────────┘
                              │
                              ▼
                        ┌──────────────┐
                        │   FastAPI    │
                        │  ML Service  │
                        └──────────────┘
```

## Backend API Reference

### Authentication

**Register User**
```javascript
POST /api/auth/register
{
  "username": "john.doe",
  "email": "john@example.com",
  "password": "SecurePass123!",
  "role": "user" // or "admin"
}
```

**Login**
```javascript
POST /api/auth/login
{
  "email": "john@example.com",
  "password": "SecurePass123!"
}

// Response includes JWT token
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": { "id": "...", "role": "user" }
}
```

### User Management (Admin)

**Get All Users**
```javascript
GET /api/users
Headers: { Authorization: "Bearer <token>" }
```

**Update User**
```javascript
PUT /api/users/:userId
Headers: { Authorization: "Bearer <token>" }
{
  "username": "updated.name",
  "role": "admin",
  "status": "active"
}
```

**Delete User**
```javascript
DELETE /api/users/:userId
Headers: { Authorization: "Bearer <token>" }
```

### Task Management

**Create Task**
```javascript
POST /api/tasks
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Implement authentication",
  "description": "Add JWT-based auth system",
  "assignedTo": "userId",
  "priority": "high",
  "dueDate": "2026-05-01",
  "status": "todo"
}
```

**Update Task Status**
```javascript
PATCH /api/tasks/:taskId/status
{
  "status": "in-progress" // or "done"
}
```

**Get User Tasks**
```javascript
GET /api/tasks/user/:userId
Headers: { Authorization: "Bearer <token>" }
```

### Support Tickets

**Create Ticket**
```javascript
POST /api/tickets
Headers: { Authorization: "Bearer <token>" }
{
  "title": "Login not working",
  "description": "Cannot login with correct credentials",
  "priority": "high",
  "category": "technical"
}
```

**Get Tickets**
```javascript
GET /api/tickets
GET /api/tickets/:ticketId
```

## Frontend Implementation

### Authentication Context

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  const login = async (email, password) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    
    const data = await response.json();
    
    if (response.ok) {
      setToken(data.token);
      setUser(data.user);
      localStorage.setItem('token', data.token);
      return { success: true };
    }
    
    return { success: false, error: data.message };
  };

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem('token');
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';
import './KanbanBoard.css';

const KanbanBoard = ({ userId, token }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const response = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
      {
        headers: { Authorization: `Bearer ${token}` }
      }
    );
    
    const data = await response.json();
    
    const organized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    
    setTasks(organized);
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    await fetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${token}`
        },
        body: JSON.stringify({ status: newStatus })
      }
    );
    
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <KanbanColumn
        title="To Do"
        tasks={tasks.todo}
        onStatusChange={updateTaskStatus}
        targetStatus="in-progress"
      />
      <KanbanColumn
        title="In Progress"
        tasks={tasks.inProgress}
        onStatusChange={updateTaskStatus}
        targetStatus="done"
      />
      <KanbanColumn
        title="Done"
        tasks={tasks.done}
        onStatusChange={updateTaskStatus}
      />
    </div>
  );
};

const KanbanColumn = ({ title, tasks, onStatusChange, targetStatus }) => (
  <div className="kanban-column">
    <h3>{title}</h3>
    {tasks.map(task => (
      <div key={task._id} className="task-card">
        <h4>{task.title}</h4>
        <p>{task.description}</p>
        <span className={`priority-${task.priority}`}>{task.priority}</span>
        {targetStatus && (
          <button onClick={() => onStatusChange(task._id, targetStatus)}>
            Move to {targetStatus}
          </button>
        )}
      </div>
    ))}
  </div>
);

export default KanbanBoard;
```

### Time Tracker Component

```javascript
// src/components/TimeTracker.js
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId, token }) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval = null;
    
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    } else {
      clearInterval(interval);
    }
    
    return () => clearInterval(interval);
  }, [isRunning]);

  const saveTimeLog = async () => {
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`
      },
      body: JSON.stringify({ duration: seconds })
    });
  };

  const formatTime = (totalSeconds) => {
    const hours = Math.floor(totalSeconds / 3600);
    const minutes = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={() => { setSeconds(0); setIsRunning(false); }}>
        Reset
      </button>
      <button onClick={saveTimeLog} disabled={seconds === 0}>
        Save
      </button>
    </div>
  );
};

export default TimeTracker;
```

## ML Service Integration

### Risk Detection

```javascript
// Call risk detection API
const detectRisk = async (userId, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-detection/${userId}`,
    {
      headers: { Authorization: `Bearer ${token}` }
    }
  );
  
  const data = await response.json();
  
  return {
    riskScore: data.risk_score, // 0-100
    riskLevel: data.risk_level, // low, medium, high
    factors: data.factors,
    recommendations: data.recommendations
  };
};
```

### Burnout Detection

```javascript
const checkBurnout = async (userId, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`
      },
      body: JSON.stringify({
        userId,
        workload: 45, // hours per week
        taskCount: 12,
        completionRate: 0.75,
        overtimeHours: 10
      })
    }
  );
  
  const data = await response.json();
  
  return {
    burnoutScore: data.burnout_score,
    status: data.status, // healthy, at-risk, burnout
    recommendations: data.recommendations
  };
};
```

### Ticket Classification

```javascript
const classifyTicket = async (ticketText, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`
      },
      body: JSON.stringify({
        title: ticketText.title,
        description: ticketText.description
      })
    }
  );
  
  const data = await response.json();
  
  return {
    category: data.category, // technical, billing, feature, other
    priority: data.priority, // low, medium, high, critical
    assignTo: data.suggested_assignee,
    confidence: data.confidence
  };
};
```

## Admin Dashboard Implementation

```javascript
// src/pages/AdminDashboard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const AdminDashboard = () => {
  const { token } = useContext(AuthContext);
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const [tickets, setTickets] = useState([]);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    // Fetch users
    const usersRes = await fetch(
      `${process.env.REACT_APP_API_URL}/api/users`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setUsers(await usersRes.json());

    // Fetch analytics
    const analyticsRes = await fetch(
      `${process.env.REACT_APP_API_URL}/api/analytics`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setAnalytics(await analyticsRes.json());

    // Fetch tickets
    const ticketsRes = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tickets`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setTickets(await ticketsRes.json());
  };

  const deleteUser = async (userId) => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      await fetch(
        `${process.env.REACT_APP_API_URL}/api/users/${userId}`,
        {
          method: 'DELETE',
          headers: { Authorization: `Bearer ${token}` }
        }
      );
      fetchDashboardData();
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      {analytics && (
        <div className="analytics-cards">
          <div className="card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="card">
            <h3>Open Tickets</h3>
            <p>{analytics.openTickets}</p>
          </div>
          <div className="card">
            <h3>High Risk Users</h3>
            <p>{analytics.highRiskUsers}</p>
          </div>
        </div>
      )}

      <div className="users-section">
        <h2>User Management</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.status}</td>
                <td>
                  <button onClick={() => editUser(user._id)}>Edit</button>
                  <button onClick={() => deleteUser(user._id)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>

      <div className="tickets-section">
        <h2>Support Tickets</h2>
        {tickets.map(ticket => (
          <div key={ticket._id} className="ticket-card">
            <h4>{ticket.title}</h4>
            <span className={`priority-${ticket.priority}`}>
              {ticket.priority}
            </span>
            <span className={`status-${ticket.status}`}>
              {ticket.status}
            </span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, token } = useContext(AuthContext);

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && user.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

export default ProtectedRoute;

// Usage in App.js
<Route
  path="/admin"
  element={
    <ProtectedRoute requiredRole="admin">
      <AdminDashboard />
    </ProtectedRoute>
  }
/>
```

### API Service Layer

```javascript
// src/services/api.js
const API_BASE = process.env.REACT_APP_API_URL;

const getAuthHeader = () => ({
  Authorization: `Bearer ${localStorage.getItem('token')}`
});

export const userService = {
  getAll: async () => {
    const res = await fetch(`${API_BASE}/api/users`, {
      headers: getAuthHeader()
    });
    return res.json();
  },
  
  update: async (userId, data) => {
    const res = await fetch(`${API_BASE}/api/users/${userId}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        ...getAuthHeader()
      },
      body: JSON.stringify(data)
    });
    return res.json();
  }
};

export const taskService = {
  getUserTasks: async (userId) => {
    const res = await fetch(`${API_BASE}/api/tasks/user/${userId}`, {
      headers: getAuthHeader()
    });
    return res.json();
  },
  
  create: async (taskData) => {
    const res = await fetch(`${API_BASE}/api/tasks`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...getAuthHeader()
      },
      body: JSON.stringify(taskData)
    });
    return res.json();
  }
};
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_strong_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
```

**ML Service (.env)**
```bash
BACKEND_URL=http://localhost:5000
MODEL_PATH=./models
LOG_LEVEL=INFO
RISK_THRESHOLD=0.7
BURNOUT_THRESHOLD=0.6
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_ANALYTICS=true
```

## Troubleshooting

### Authentication Issues

**Problem**: JWT token expired
```javascript
// Add token refresh logic
const refreshToken = async () => {
  const response = await fetch(`${API_BASE}/api/auth/refresh`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${localStorage.getItem('token')}`
    }
  });
  
  if (response.ok) {
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data.token;
  }
  
  // Logout if refresh fails
  localStorage.removeItem('token');
  window.location.href = '/login';
};
```

### MongoDB Connection Issues

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Not Responding

```bash
# Check ML service logs
tail -f ml-service/logs/app.log

# Verify dependencies
pip list | grep -E "fastapi|scikit-learn|river"

# Test ML endpoint directly
curl -X POST http://localhost:8000/api/ml/health
```

### Task Status Not Updating

```javascript
// Ensure proper state management
const [tasks, setTasks] = useState([]);
const [loading, setLoading] = useState(false);

const updateTaskStatus = async (taskId, status) => {
  setLoading(true);
  
  try {
    const response = await fetch(
      `${API_BASE}/api/tasks/${taskId}/status`,
      {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${token}`
        },
        body: JSON.stringify({ status })
      }
    );
    
    if (!response.ok) {
      throw new Error('Failed to update task');
    }
    
    // Refresh tasks
    await fetchTasks();
  } catch (error) {
    console.error('Update error:', error);
    alert('Failed to update task status');
  } finally {
    setLoading(false);
  }
};
```

## Deployment

### Production Build

```bash
# Backend
cd backend
npm run build
NODE_ENV=production npm start

# Frontend
cd frontend
npm run build
# Deploy build/ folder to hosting service

# ML Service
cd ml-service
pip install -r requirements.txt
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app
```

### Environment Variables for Production

Use secure environment variable management (not hardcoded):

```bash
# Use AWS Secrets Manager, Azure Key Vault, or similar
export MONGODB_URI=$(aws secretsmanager get-secret-value --secret-id prod/mongodb)
export JWT_SECRET=$(aws secretsmanager get-secret-value --secret-id prod/jwt)
```

This skill provides comprehensive coverage for implementing and working with the Enterprise User Management System with AI Analytics.
