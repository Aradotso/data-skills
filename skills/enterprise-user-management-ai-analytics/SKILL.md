---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task management, ticket classification, and risk detection
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - create a user management dashboard with AI
  - implement ticket classification and risk detection
  - build an admin dashboard with anomaly detection
  - set up JWT authentication for user management
  - use AI for burnout detection and task analytics
  - deploy enterprise user management with FastAPI ML service
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user/task management with AI-driven insights. The system provides role-based access control, Kanban task tracking, support ticket management, and ML-powered features including risk prediction, anomaly detection, burnout analysis, and ticket classification.

**Architecture:**
- **Frontend:** React.js (port 3000)
- **Backend:** Node.js with Express (port 5000)
- **ML Service:** FastAPI with scikit-learn and River (port 8000)
- **Database:** MongoDB
- **Auth:** JWT tokens

## Installation

### Prerequisites

```bash
# Required software
node --version  # v14+
npm --version   # v6+
python --version  # v3.8+
mongod --version  # v4.4+
```

### Full Stack Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
npm start  # Runs on http://localhost:5000

# ML Service setup (new terminal)
cd ml-service
pip install -r requirements.txt
uvicorn main:app --reload  # Runs on http://localhost:8000

# Frontend setup (new terminal)
cd frontend
npm install
npm start  # Runs on http://localhost:3000
```

### Environment Configuration

**Backend (.env):**
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service (.env):**
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
```

**Frontend (.env):**
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

## Backend API Usage

### Authentication

```javascript
// Register new user
const register = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user' // 'user' or 'admin'
    })
  });
  return response.json();
};

// Login
const login = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: credentials.email,
      password: credentials.password
    })
  });
  const data = await response.json();
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};

// Authenticated request helper
const authFetch = async (url, options = {}) => {
  const token = localStorage.getItem('token');
  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async () => {
  const response = await authFetch('http://localhost:5000/api/users');
  return response.json();
};

// Create user
const createUser = async (userData) => {
  const response = await authFetch('http://localhost:5000/api/users', {
    method: 'POST',
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      role: userData.role,
      department: userData.department
    })
  });
  return response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const response = await authFetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    body: JSON.stringify(updates)
  });
  return response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const response = await authFetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE'
  });
  return response.json();
};
```

### Task Management

```javascript
// Get user tasks
const getUserTasks = async (userId) => {
  const response = await authFetch(`http://localhost:5000/api/tasks/user/${userId}`);
  return response.json();
};

// Create task
const createTask = async (taskData) => {
  const response = await authFetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: taskData.status || 'todo', // 'todo', 'in-progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Update task status (Kanban)
const updateTaskStatus = async (taskId, status) => {
  const response = await authFetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    body: JSON.stringify({ status })
  });
  return response.json();
};

// Track time on task
const trackTime = async (taskId, timeData) => {
  const response = await authFetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    body: JSON.stringify({
      duration: timeData.duration, // in seconds
      date: new Date().toISOString()
    })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const response = await authFetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// Get user tickets
const getUserTickets = async () => {
  const response = await authFetch('http://localhost:5000/api/tickets/my-tickets');
  return response.json();
};

// Update ticket status
const updateTicketStatus = async (ticketId, status) => {
  const response = await authFetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    body: JSON.stringify({ status })
  });
  return response.json();
};
```

## ML Service API Usage

### Ticket Classification

```javascript
// Classify ticket using AI
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketText.title,
      description: ticketText.description
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.85 }
  return result;
};
```

### Risk Prediction

```javascript
// Predict user risk based on behavior
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        failed_login_attempts: 3,
        unusual_activity: true,
        data_access_frequency: 45,
        after_hours_access: 12
      }
    })
  });
  const result = await response.json();
  // Returns: { risk_score: 0.72, risk_level: 'medium', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomalies = async (behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: behaviorData.userId,
      metrics: {
        login_time: behaviorData.loginTime,
        location: behaviorData.location,
        device_type: behaviorData.deviceType,
        activity_pattern: behaviorData.activityPattern
      }
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.89, explanation: '...' }
  return result;
};
```

### Burnout Detection

```javascript
// Analyze user workload for burnout risk
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      workload_data: {
        hours_worked_weekly: 65,
        tasks_completed: 45,
        tasks_pending: 23,
        overtime_hours: 25,
        days_since_break: 14
      }
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.78, recommendations: [...] }
  return result;
};
```

### Predictive Project Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.projectId,
      features: {
        tasks_total: 50,
        tasks_completed: 12,
        days_elapsed: 15,
        days_remaining: 30,
        team_size: 5,
        avg_task_completion_time: 2.5
      }
    })
  });
  const result = await response.json();
  // Returns: { delay_probability: 0.65, estimated_delay_days: 7, suggestions: [...] }
  return result;
};
```

## React Frontend Integration

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [burnoutRisk, setBurnoutRisk] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    try {
      const userId = JSON.parse(localStorage.getItem('user')).id;
      
      // Fetch tasks
      const tasksRes = await authFetch(`${process.env.REACT_APP_API_URL}/tasks/user/${userId}`);
      const tasksData = await tasksRes.json();
      
      // Organize by status
      setTasks({
        todo: tasksData.filter(t => t.status === 'todo'),
        inProgress: tasksData.filter(t => t.status === 'in-progress'),
        done: tasksData.filter(t => t.status === 'done')
      });

      // Check burnout risk
      const burnoutRes = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId })
      });
      const burnoutData = await burnoutRes.json();
      setBurnoutRisk(burnoutData);
      
      setLoading(false);
    } catch (error) {
      console.error('Dashboard load error:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    await authFetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`, {
      method: 'PATCH',
      body: JSON.stringify({ status: newStatus })
    });
    loadDashboardData();
  };

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      {burnoutRisk && burnoutRisk.burnout_risk === 'high' && (
        <div className="alert alert-warning">
          ⚠️ High burnout risk detected. Consider taking a break.
        </div>
      )}
      
      <div className="kanban-board">
        <div className="column">
          <h3>To Do ({tasks.todo.length})</h3>
          {tasks.todo.map(task => (
            <TaskCard 
              key={task.id} 
              task={task} 
              onMove={() => moveTask(task.id, 'in-progress')}
            />
          ))}
        </div>
        
        <div className="column">
          <h3>In Progress ({tasks.inProgress.length})</h3>
          {tasks.inProgress.map(task => (
            <TaskCard 
              key={task.id} 
              task={task} 
              onMove={() => moveTask(task.id, 'done')}
            />
          ))}
        </div>
        
        <div className="column">
          <h3>Done ({tasks.done.length})</h3>
          {tasks.done.map(task => (
            <TaskCard key={task.id} task={task} />
          ))}
        </div>
      </div>
    </div>
  );
};
```

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    // Get all users
    const usersRes = await authFetch(`${process.env.REACT_APP_API_URL}/users`);
    const users = await usersRes.json();

    // Check risk for each user
    const riskChecks = await Promise.all(
      users.map(async (user) => {
        const res = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ user_id: user.id })
        });
        const risk = await res.json();
        return { ...user, risk };
      })
    );

    const highRisk = riskChecks.filter(u => u.risk.risk_level === 'high');
    setRiskUsers(highRisk);

    // Get system analytics
    const analyticsRes = await authFetch(`${process.env.REACT_APP_API_URL}/analytics/overview`);
    setAnalytics(await analyticsRes.json());
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Analytics</h1>
      
      {analytics && (
        <div className="stats-grid">
          <div className="stat-card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="stat-card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="stat-card">
            <h3>Open Tickets</h3>
            <p>{analytics.openTickets}</p>
          </div>
        </div>
      )}

      {riskUsers.length > 0 && (
        <div className="risk-alerts">
          <h2>⚠️ High Risk Users</h2>
          {riskUsers.map(user => (
            <div key={user.id} className="risk-card">
              <h4>{user.username}</h4>
              <p>Risk Score: {user.risk.risk_score.toFixed(2)}</p>
              <p>Factors: {user.risk.factors.join(', ')}</p>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};
```

### Smart Ticket Creation with AI

```javascript
import React, { useState } from 'react';

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: ''
  });
  const [aiSuggestion, setAiSuggestion] = useState(null);

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const classifyWithAI = async () => {
    if (!formData.title || !formData.description) return;

    const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        title: formData.title,
        description: formData.description
      })
    });
    
    const classification = await response.json();
    setAiSuggestion(classification);
  };

  const submitTicket = async () => {
    const ticketData = {
      ...formData,
      category: aiSuggestion?.category || 'general',
      priority: aiSuggestion?.priority || 'medium'
    };

    await authFetch(`${process.env.REACT_APP_API_URL}/tickets`, {
      method: 'POST',
      body: JSON.stringify(ticketData)
    });

    alert('Ticket created successfully!');
    setFormData({ title: '', description: '' });
    setAiSuggestion(null);
  };

  return (
    <div className="create-ticket">
      <h2>Create Support Ticket</h2>
      
      <input
        type="text"
        name="title"
        placeholder="Title"
        value={formData.title}
        onChange={handleChange}
      />
      
      <textarea
        name="description"
        placeholder="Description"
        value={formData.description}
        onChange={handleChange}
        rows="5"
      />

      <button onClick={classifyWithAI}>
        🤖 Classify with AI
      </button>

      {aiSuggestion && (
        <div className="ai-suggestion">
          <h4>AI Suggestions</h4>
          <p><strong>Category:</strong> {aiSuggestion.category}</p>
          <p><strong>Priority:</strong> {aiSuggestion.priority}</p>
          <p><strong>Confidence:</strong> {(aiSuggestion.confidence * 100).toFixed(1)}%</p>
        </div>
      )}

      <button onClick={submitTicket} disabled={!aiSuggestion}>
        Submit Ticket
      </button>
    </div>
  );
};
```

## Common Patterns

### Role-Based Access Control

```javascript
// Middleware for protected routes
const requireAuth = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

const requireAdmin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

// Usage in routes
app.get('/api/users', requireAuth, requireAdmin, getAllUsers);
app.get('/api/tasks/my-tasks', requireAuth, getMyTasks);
```

### Real-time Notifications

```javascript
// Socket.io integration for live updates
const setupNotifications = (io) => {
  io.on('connection', (socket) => {
    socket.on('join-user-room', (userId) => {
      socket.join(`user-${userId}`);
    });

    // Emit notification when task is assigned
    socket.on('task-assigned', (data) => {
      io.to(`user-${data.userId}`).emit('notification', {
        type: 'task_assigned',
        message: `New task assigned: ${data.taskTitle}`,
        timestamp: new Date()
      });
    });

    // Emit when risk detected
    socket.on('risk-detected', (data) => {
      io.to(`user-${data.userId}`).emit('alert', {
        type: 'security',
        level: 'high',
        message: 'Unusual activity detected on your account'
      });
    });
  });
};
```

### Audit Logging

```javascript
// Log all critical actions
const logAction = async (userId, action, details) => {
  await authFetch(`${process.env.REACT_APP_API_URL}/audit-logs`, {
    method: 'POST',
    body: JSON.stringify({
      user_id: userId,
      action: action,
      details: details,
      timestamp: new Date().toISOString(),
      ip_address: window.location.hostname
    })
  });
};

// Usage
await logAction(user.id, 'USER_DELETED', { deletedUserId: targetUser.id });
await logAction(user.id, 'TASK_CREATED', { taskId: newTask.id, assignedTo: newTask.assignedTo });
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection in backend
# Ensure MONGODB_URI in .env matches your setup
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
```

### JWT Token Expiration

```javascript
// Implement token refresh
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/refresh`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};

// Add interceptor to handle 401 errors
const authFetch = async (url, options = {}) => {
  let response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${localStorage.getItem('token')}`
    }
  });

  if (response.status === 401) {
    await refreshToken();
    response = await fetch(url, options); // Retry with new token
  }

  return response;
};
```

### ML Service Performance

```python
# If predictions are slow, cache common requests
from functools import lru_cache

@lru_cache(maxsize=100)
def get_user_risk_cached(user_id: str):
    # Cache risk scores for 5 minutes
    return calculate_risk(user_id)

# Use batch predictions for multiple users
async def batch_risk_prediction(user_ids: list):
    results = []
    for batch in chunks(user_ids, 10):
        batch_results = await asyncio.gather(*[
            predict_risk(uid) for uid in batch
        ])
        results.extend(batch_results)
    return results
```

### CORS Errors

```javascript
// Backend CORS configuration
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Task Status Not Updating

```javascript
// Ensure proper state management
const updateTaskStatus = async (taskId, newStatus) => {
  try {
    const response = await authFetch(
      `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
      {
        method: 'PATCH',
        body: JSON.stringify({ status: newStatus })
      }
    );

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const updated = await response.json();
    
    // Update local state
    setTasks(prevTasks => ({
      ...prevTasks,
      [newStatus]: [...prevTasks[newStatus], updated],
      [oldStatus]: prevTasks[oldStatus].filter(t => t.id !== taskId)
    }));

    return updated;
  } catch (error) {
    console.error('Failed to update task:', error);
    throw error;
  }
};
```

This skill provides comprehensive coverage for integrating and using the Enterprise User Management System with AI Analytics in full-stack applications.
