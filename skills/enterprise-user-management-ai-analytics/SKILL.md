---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, anomaly monitoring, and predictive insights
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "configure user roles and permissions in this system"
  - "implement JWT authentication for the user management app"
  - "create AI-powered ticket classification"
  - "set up the ML service for risk prediction"
  - "build a user dashboard with task tracking"
  - "configure MongoDB for enterprise user data"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack JavaScript application that combines user/task management with AI-powered analytics. It provides centralized user administration, task tracking via Kanban boards, support ticket management, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and project delay forecasting.

**Key Components:**
- **Frontend:** React.js application with user/admin dashboards
- **Backend:** Node.js REST API with JWT authentication
- **ML Service:** FastAPI-based service using scikit-learn and River for real-time predictions
- **Database:** MongoDB for user, task, and ticket storage

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB instance (local or cloud)

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
```env
PORT=5000
MONGO_URI=mongodb://localhost:27017/enterprise-users
JWT_SECRET=your_jwt_secret_key_here
ML_SERVICE_URL=http://localhost:8000
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

Create `ml-service/.env`:
```env
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

Create `frontend/.env`:
```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:
```bash
npm start
```

## Backend API Reference

### Authentication

**Register User**
```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
    })
  });
  return response.json();
};
```

**Login**
```javascript
// POST /api/auth/login
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
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management (Admin)

**Get All Users**
```javascript
// GET /api/users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

**Update User Role**
```javascript
// PUT /api/users/:id
const updateUserRole = async (userId, newRole) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ role: newRole })
  });
  return response.json();
};
```

**Delete User**
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
  return response.json();
};
```

### Task Management

**Create Task**
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
      assignedTo: taskData.userId,
      status: 'todo', // 'todo', 'inProgress', 'done'
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};
```

**Get User Tasks**
```javascript
// GET /api/tasks/user/:userId
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

**Update Task Status**
```javascript
// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Tickets

**Create Ticket**
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
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category // Will be auto-classified by AI
    })
  });
  return response.json();
};
```

**Get All Tickets (Admin)**
```javascript
// GET /api/tickets
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

## ML Service API Reference

### Risk Prediction

```javascript
// POST /api/ml/predict-risk
const predictUserRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      failedLogins: userData.failedLogins,
      tasksOverdue: userData.tasksOverdue,
      avgResponseTime: userData.avgResponseTime,
      activityScore: userData.activityScore
    })
  });
  const result = await response.json();
  // Returns: { riskLevel: 'low'|'medium'|'high', confidence: 0.85 }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /api/ml/detect-anomaly
const detectAnomaly = async (behaviorData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: behaviorData.userId,
      loginTime: behaviorData.loginTime,
      location: behaviorData.location,
      deviceFingerprint: behaviorData.deviceFingerprint,
      actionsPerformed: behaviorData.actionsPerformed
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.92 }
  return result;
};
```

### Burnout Detection

```javascript
// POST /api/ml/detect-burnout
const detectBurnout = async (workloadData) => {
  const response = await fetch('http://localhost:8000/api/ml/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: workloadData.userId,
      activeTasks: workloadData.activeTasks,
      avgWorkHours: workloadData.avgWorkHours,
      overtimeHours: workloadData.overtimeHours,
      tasksCompleted: workloadData.tasksCompleted,
      missedDeadlines: workloadData.missedDeadlines
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'high', recommendation: 'Reduce workload' }
  return result;
};
```

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
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
  // Returns: { category: 'technical', priority: 'high', assignTo: 'IT_DEPT' }
  return result;
};
```

### Project Delay Prediction

```javascript
// POST /api/ml/predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      avgTaskDuration: projectData.avgTaskDuration,
      teamSize: projectData.teamSize,
      complexity: projectData.complexity
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.73, estimatedDelay: 5 }
  return result;
};
```

## Frontend Patterns

### React Context for Authentication

```javascript
// src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      fetchUserProfile(token);
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUserProfile = async (token) => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/me', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const userData = await response.json();
      setUser(userData);
    } catch (error) {
      console.error('Auth error:', error);
      localStorage.removeItem('token');
    } finally {
      setLoading(false);
    }
  };

  const login = async (credentials) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    setUser(data.user);
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
// src/components/KanbanBoard.js
import React, { useState, useEffect } from 'react';

const KanbanBoard = ({ userId }) => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, [userId]);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const categorized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'inProgress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ status: newStatus })
    });
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={status} 
                onChange={(e) => moveTask(task._id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="inProgress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
          ))}
        </div>
      ))}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// src/components/AdminAnalytics.js
import React, { useState, useEffect } from 'react';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    riskUsers: [],
    burnoutUsers: [],
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch users
    const usersRes = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const users = await usersRes.json();

    // Get AI predictions for each user
    const predictions = await Promise.all(
      users.map(async (user) => {
        const riskRes = await fetch('http://localhost:8000/api/ml/predict-risk', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            userId: user._id,
            failedLogins: user.failedLogins || 0,
            tasksOverdue: user.tasksOverdue || 0,
            avgResponseTime: user.avgResponseTime || 0,
            activityScore: user.activityScore || 100
          })
        });
        const risk = await riskRes.json();

        const burnoutRes = await fetch('http://localhost:8000/api/ml/detect-burnout', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            userId: user._id,
            activeTasks: user.activeTasks || 0,
            avgWorkHours: user.avgWorkHours || 8,
            overtimeHours: user.overtimeHours || 0,
            tasksCompleted: user.tasksCompleted || 0,
            missedDeadlines: user.missedDeadlines || 0
          })
        });
        const burnout = await burnoutRes.json();

        return { user, risk, burnout };
      })
    );

    setAnalytics({
      riskUsers: predictions.filter(p => p.risk.riskLevel === 'high'),
      burnoutUsers: predictions.filter(p => p.burnout.burnoutRisk === 'high'),
      anomalies: [] // Populated from separate anomaly detection
    });
  };

  return (
    <div className="analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <div className="risk-alerts">
        <h3>High Risk Users ({analytics.riskUsers.length})</h3>
        {analytics.riskUsers.map(({ user, risk }) => (
          <div key={user._id} className="alert-card">
            <p>{user.username} - Risk: {risk.riskLevel}</p>
            <p>Confidence: {(risk.confidence * 100).toFixed(0)}%</p>
          </div>
        ))}
      </div>

      <div className="burnout-alerts">
        <h3>Burnout Risk ({analytics.burnoutUsers.length})</h3>
        {analytics.burnoutUsers.map(({ user, burnout }) => (
          <div key={user._id} className="alert-card">
            <p>{user.username}</p>
            <p>{burnout.recommendation}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Configuration

### MongoDB Schema Example

```javascript
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  failedLogins: { type: Number, default: 0 },
  tasksOverdue: { type: Number, default: 0 },
  activeTasks: { type: Number, default: 0 },
  avgWorkHours: { type: Number, default: 8 },
  overtimeHours: { type: Number, default: 0 },
  tasksCompleted: { type: Number, default: 0 },
  missedDeadlines: { type: Number, default: 0 },
  activityScore: { type: Number, default: 100 },
  createdAt: { type: Date, default: Date.now },
  lastLogin: { type: Date }
});

module.exports = mongoose.model('User', userSchema);
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
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

### CORS Issues

If frontend can't connect to backend:

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### JWT Token Expiration

```javascript
// Refresh token before API call
const getValidToken = async () => {
  const token = localStorage.getItem('token');
  try {
    const decoded = jwt_decode(token);
    if (decoded.exp * 1000 < Date.now()) {
      // Token expired, logout
      localStorage.removeItem('token');
      window.location.href = '/login';
      return null;
    }
    return token;
  } catch {
    return null;
  }
};
```

### ML Service Connection Issues

```javascript
// Add fallback when ML service unavailable
const predictWithFallback = async (endpoint, data) => {
  try {
    const response = await fetch(`http://localhost:8000${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    return await response.json();
  } catch (error) {
    console.warn('ML service unavailable, using default');
    return { riskLevel: 'unknown', message: 'ML service offline' };
  }
};
```

### Database Connection

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

## Common Patterns

### Protected Routes

```javascript
// src/components/ProtectedRoute.js
import { Navigate } from 'react-router-dom';
import { useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) return <div>Loading...</div>;
  if (!user) return <Navigate to="/login" />;
  if (adminOnly && user.role !== 'admin') return <Navigate to="/dashboard" />;

  return children;
};

export default ProtectedRoute;
```

### Real-time Notifications

```javascript
// src/hooks/useNotifications.js
import { useState, useEffect } from 'react';

export const useNotifications = (userId) => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const interval = setInterval(async () => {
      const token = localStorage.getItem('token');
      const response = await fetch(`http://localhost:5000/api/notifications/${userId}`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setNotifications(data);
    }, 30000); // Poll every 30 seconds

    return () => clearInterval(interval);
  }, [userId]);

  return notifications;
};
```
