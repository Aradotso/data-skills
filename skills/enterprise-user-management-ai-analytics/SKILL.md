---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "create admin dashboard with task tracking"
  - "implement JWT authentication for user system"
  - "build kanban board with AI insights"
  - "add AI ticket classification and risk detection"
  - "configure user management with ML service"
  - "deploy enterprise management system with analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform combining React frontend, Node.js backend, and FastAPI ML service. Provides role-based access control, task management with Kanban boards, AI-powered ticket classification, risk prediction, anomaly detection, and burnout analysis.

## What This Project Does

- **User Management**: JWT-authenticated system with admin and user roles
- **Task Tracking**: Kanban boards (To Do → In Progress → Done) with time tracking
- **Support Tickets**: AI-classified ticket routing and management
- **AI Analytics**: Risk detection, anomaly detection, burnout analysis, and predictive insights
- **Admin Dashboard**: User management, task assignment, audit logs, and organization analytics

## Installation

### Prerequisites

```bash
# Node.js 14+ for backend and frontend
# Python 3.8+ for ML service
# MongoDB instance (local or cloud)
```

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

Create `.env` file in backend directory:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

Start backend:

```bash
npm start
# Runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
# Runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Runs at http://localhost:3000
```

## Backend API (Node.js)

### Authentication

```javascript
// Register new user
const axios = require('axios');

const registerUser = async (userData) => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/register', {
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user' // 'user' or 'admin'
    });
    return response.data;
  } catch (error) {
    console.error('Registration error:', error.response.data);
  }
};

// Login
const loginUser = async (email, password) => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
      email,
      password
    });
    // Store token
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    return { token, user };
  } catch (error) {
    console.error('Login error:', error.response.data);
  }
};
```

### User Management (Admin)

```javascript
// Get all users (admin only)
const getAllUsers = async (token) => {
  try {
    const response = await axios.get('http://localhost:5000/api/users', {
      headers: { Authorization: `Bearer ${token}` }
    });
    return response.data;
  } catch (error) {
    console.error('Fetch users error:', error.response.data);
  }
};

// Update user
const updateUser = async (userId, updates, token) => {
  try {
    const response = await axios.put(
      `http://localhost:5000/api/users/${userId}`,
      updates,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    return response.data;
  } catch (error) {
    console.error('Update user error:', error.response.data);
  }
};

// Delete user
const deleteUser = async (userId, token) => {
  try {
    await axios.delete(`http://localhost:5000/api/users/${userId}`, {
      headers: { Authorization: `Bearer ${token}` }
    });
  } catch (error) {
    console.error('Delete user error:', error.response.data);
  }
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData, token) => {
  try {
    const response = await axios.post(
      'http://localhost:5000/api/tasks',
      {
        title: taskData.title,
        description: taskData.description,
        assignedTo: taskData.userId,
        priority: taskData.priority, // 'low', 'medium', 'high'
        status: 'todo', // 'todo', 'inprogress', 'done'
        dueDate: taskData.dueDate
      },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    return response.data;
  } catch (error) {
    console.error('Create task error:', error.response.data);
  }
};

// Get user tasks
const getUserTasks = async (userId, token) => {
  try {
    const response = await axios.get(
      `http://localhost:5000/api/tasks/user/${userId}`,
      { headers: { Authorization: `Bearer ${token}` } }
    );
    return response.data;
  } catch (error) {
    console.error('Fetch tasks error:', error.response.data);
  }
};

// Update task status
const updateTaskStatus = async (taskId, status, token) => {
  try {
    const response = await axios.patch(
      `http://localhost:5000/api/tasks/${taskId}/status`,
      { status }, // 'todo', 'inprogress', 'done'
      { headers: { Authorization: `Bearer ${token}` } }
    );
    return response.data;
  } catch (error) {
    console.error('Update task status error:', error.response.data);
  }
};

// Track time on task
const trackTaskTime = async (taskId, timeSpent, token) => {
  try {
    const response = await axios.post(
      `http://localhost:5000/api/tasks/${taskId}/time`,
      { timeSpent }, // in minutes
      { headers: { Authorization: `Bearer ${token}` } }
    );
    return response.data;
  } catch (error) {
    console.error('Track time error:', error.response.data);
  }
};
```

### Ticket Management

```javascript
// Create support ticket
const createTicket = async (ticketData, token) => {
  try {
    const response = await axios.post(
      'http://localhost:5000/api/tickets',
      {
        title: ticketData.title,
        description: ticketData.description,
        priority: ticketData.priority,
        category: ticketData.category // 'technical', 'billing', 'general'
      },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    return response.data;
  } catch (error) {
    console.error('Create ticket error:', error.response.data);
  }
};

// Get all tickets
const getAllTickets = async (token) => {
  try {
    const response = await axios.get('http://localhost:5000/api/tickets', {
      headers: { Authorization: `Bearer ${token}` }
    });
    return response.data;
  } catch (error) {
    console.error('Fetch tickets error:', error.response.data);
  }
};

// Update ticket status
const updateTicketStatus = async (ticketId, status, token) => {
  try {
    const response = await axios.patch(
      `http://localhost:5000/api/tickets/${ticketId}`,
      { status }, // 'open', 'in_progress', 'resolved', 'closed'
      { headers: { Authorization: `Bearer ${token}` } }
    );
    return response.data;
  } catch (error) {
    console.error('Update ticket error:', error.response.data);
  }
};
```

## ML Service API (FastAPI)

### AI Ticket Classification

```python
import requests

# Classify ticket using AI
def classify_ticket(ticket_text, ticket_id):
    url = "http://localhost:8000/api/ml/classify-ticket"
    payload = {
        "ticket_id": ticket_id,
        "text": ticket_text,
        "title": "Ticket Title"
    }
    response = requests.post(url, json=payload)
    return response.json()

# Example
result = classify_ticket(
    "Cannot access my account after password reset",
    "ticket_123"
)
print(result)
# {
#   "category": "technical",
#   "priority": "high",
#   "confidence": 0.87,
#   "suggested_assignee": "support_team_a"
# }
```

### Risk Prediction

```python
# Predict user risk based on behavior
def predict_user_risk(user_id, user_data):
    url = "http://localhost:8000/api/ml/risk-prediction"
    payload = {
        "user_id": user_id,
        "login_frequency": user_data["login_frequency"],
        "failed_logins": user_data["failed_logins"],
        "task_completion_rate": user_data["task_completion_rate"],
        "access_pattern_anomaly": user_data["access_pattern_anomaly"]
    }
    response = requests.post(url, json=payload)
    return response.json()

# Example
risk_data = predict_user_risk("user_456", {
    "login_frequency": 15,
    "failed_logins": 5,
    "task_completion_rate": 0.65,
    "access_pattern_anomaly": 0.3
})
print(risk_data)
# {
#   "risk_level": "medium",
#   "risk_score": 0.62,
#   "factors": ["failed_logins", "task_completion_rate"],
#   "recommendation": "Monitor user activity closely"
# }
```

### Anomaly Detection

```python
# Detect anomalies in user behavior
def detect_anomalies(user_id, activity_data):
    url = "http://localhost:8000/api/ml/anomaly-detection"
    payload = {
        "user_id": user_id,
        "activities": activity_data
    }
    response = requests.post(url, json=payload)
    return response.json()

# Example
anomaly_result = detect_anomalies("user_789", {
    "login_time": "03:00",
    "location": "Unknown",
    "actions_per_minute": 50,
    "data_access_volume": 1000
})
print(anomaly_result)
# {
#   "is_anomaly": true,
#   "anomaly_score": 0.85,
#   "anomalies_detected": ["unusual_login_time", "high_action_rate"],
#   "alert_level": "high"
# }
```

### Burnout Detection

```python
# Analyze workload for burnout risk
def detect_burnout(user_id, workload_data):
    url = "http://localhost:8000/api/ml/burnout-detection"
    payload = {
        "user_id": user_id,
        "tasks_assigned": workload_data["tasks_assigned"],
        "tasks_completed": workload_data["tasks_completed"],
        "avg_working_hours": workload_data["avg_working_hours"],
        "overtime_hours": workload_data["overtime_hours"],
        "stress_indicators": workload_data["stress_indicators"]
    }
    response = requests.post(url, json=payload)
    return response.json()

# Example
burnout_analysis = detect_burnout("user_321", {
    "tasks_assigned": 35,
    "tasks_completed": 28,
    "avg_working_hours": 11.5,
    "overtime_hours": 20,
    "stress_indicators": 0.75
})
print(burnout_analysis)
# {
#   "burnout_risk": "high",
#   "burnout_score": 0.78,
#   "contributing_factors": ["excessive_overtime", "high_task_load"],
#   "recommendation": "Reduce workload and provide support"
# }
```

### Predictive Project Insights

```python
# Predict project delays
def predict_project_delay(project_id, project_data):
    url = "http://localhost:8000/api/ml/project-prediction"
    payload = {
        "project_id": project_id,
        "tasks_total": project_data["tasks_total"],
        "tasks_completed": project_data["tasks_completed"],
        "days_remaining": project_data["days_remaining"],
        "team_size": project_data["team_size"],
        "complexity_score": project_data["complexity_score"]
    }
    response = requests.post(url, json=payload)
    return response.json()

# Example
prediction = predict_project_delay("proj_001", {
    "tasks_total": 50,
    "tasks_completed": 20,
    "days_remaining": 10,
    "team_size": 5,
    "complexity_score": 0.8
})
print(prediction)
# {
#   "delay_probability": 0.72,
#   "estimated_delay_days": 5,
#   "completion_probability": 0.28,
#   "recommendations": ["Add resources", "Reprioritize tasks"]
# }
```

## Frontend Integration (React)

### Authentication Hook

```javascript
// hooks/useAuth.js
import { useState, useEffect } from 'react';
import axios from 'axios';

export const useAuth = () => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    if (token) {
      // Verify token and get user data
      axios.get(`${API_URL}/api/auth/me`, {
        headers: { Authorization: `Bearer ${token}` }
      })
      .then(res => setUser(res.data))
      .catch(() => logout());
    }
  }, [token]);

  const login = async (email, password) => {
    const response = await axios.post(`${API_URL}/api/auth/login`, {
      email,
      password
    });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    setToken(token);
    setUser(user);
    return user;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return { user, token, login, logout, isAuthenticated: !!token };
};
```

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useAuth } from '../hooks/useAuth';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inprogress: [], done: [] });
  const { token, user } = useAuth();
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(
        `${API_URL}/api/tasks/user/${user.id}`,
        { headers: { Authorization: `Bearer ${token}` } }
      );
      
      const grouped = response.data.reduce((acc, task) => {
        acc[task.status].push(task);
        return acc;
      }, { todo: [], inprogress: [], done: [] });
      
      setTasks(grouped);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${API_URL}/api/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchTasks();
    } catch (error) {
      console.error('Error moving task:', error);
    }
  };

  const renderColumn = (status, title) => (
    <div className="kanban-column">
      <h3>{title}</h3>
      {tasks[status].map(task => (
        <div key={task.id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>{task.priority}</span>
          {status !== 'done' && (
            <button onClick={() => moveTask(task.id, 
              status === 'todo' ? 'inprogress' : 'done'
            )}>
              Move →
            </button>
          )}
        </div>
      ))}
    </div>
  );

  return (
    <div className="kanban-board">
      {renderColumn('todo', 'To Do')}
      {renderColumn('inprogress', 'In Progress')}
      {renderColumn('done', 'Done')}
    </div>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalytics.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useAuth } from '../hooks/useAuth';

const AIAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const { token, user } = useAuth();
  const ML_URL = process.env.REACT_APP_ML_URL;

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Fetch multiple AI insights
      const [riskData, burnoutData, anomalyData] = await Promise.all([
        axios.post(`${ML_URL}/api/ml/risk-prediction`, {
          user_id: user.id,
          login_frequency: user.loginStats.frequency,
          failed_logins: user.loginStats.failed,
          task_completion_rate: user.taskStats.completionRate,
          access_pattern_anomaly: user.behaviorScore
        }),
        axios.post(`${ML_URL}/api/ml/burnout-detection`, {
          user_id: user.id,
          tasks_assigned: user.taskStats.assigned,
          tasks_completed: user.taskStats.completed,
          avg_working_hours: user.workStats.avgHours,
          overtime_hours: user.workStats.overtime,
          stress_indicators: user.stressScore
        }),
        axios.post(`${ML_URL}/api/ml/anomaly-detection`, {
          user_id: user.id,
          activities: user.recentActivities
        })
      ]);

      setAnalytics({
        risk: riskData.data,
        burnout: burnoutData.data,
        anomaly: anomalyData.data
      });
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  if (!analytics) return <div>Loading analytics...</div>;

  return (
    <div className="ai-analytics">
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level risk-${analytics.risk.risk_level}`}>
          {analytics.risk.risk_level.toUpperCase()}
        </div>
        <p>Score: {(analytics.risk.risk_score * 100).toFixed(1)}%</p>
        <p>{analytics.risk.recommendation}</p>
      </div>

      <div className="analytics-card">
        <h3>Burnout Detection</h3>
        <div className={`burnout-risk burnout-${analytics.burnout.burnout_risk}`}>
          {analytics.burnout.burnout_risk.toUpperCase()} RISK
        </div>
        <p>Score: {(analytics.burnout.burnout_score * 100).toFixed(1)}%</p>
        <ul>
          {analytics.burnout.contributing_factors.map((factor, i) => (
            <li key={i}>{factor}</li>
          ))}
        </ul>
      </div>

      <div className="analytics-card">
        <h3>Anomaly Detection</h3>
        {analytics.anomaly.is_anomaly ? (
          <>
            <div className="alert-badge">ANOMALY DETECTED</div>
            <p>Score: {(analytics.anomaly.anomaly_score * 100).toFixed(1)}%</p>
            <ul>
              {analytics.anomaly.anomalies_detected.map((anomaly, i) => (
                <li key={i}>{anomaly}</li>
              ))}
            </ul>
          </>
        ) : (
          <p>No anomalies detected</p>
        )}
      </div>
    </div>
  );
};

export default AIAnalytics;
```

## Common Patterns

### Protected Route

```javascript
// components/ProtectedRoute.jsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { isAuthenticated, user } = useAuth();

  if (!isAuthenticated) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Admin User Management

```javascript
// components/AdminUserManagement.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import { useAuth } from '../hooks/useAuth';

const AdminUserManagement = () => {
  const [users, setUsers] = useState([]);
  const { token } = useAuth();
  const API_URL = process.env.REACT_APP_API_URL;

  useEffect(() => {
    fetchUsers();
  }, []);

  const fetchUsers = async () => {
    try {
      const response = await axios.get(`${API_URL}/api/users`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      setUsers(response.data);
    } catch (error) {
      console.error('Error fetching users:', error);
    }
  };

  const updateUserRole = async (userId, newRole) => {
    try {
      await axios.put(
        `${API_URL}/api/users/${userId}`,
        { role: newRole },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUsers();
    } catch (error) {
      console.error('Error updating user:', error);
    }
  };

  const deleteUser = async (userId) => {
    if (!window.confirm('Are you sure?')) return;
    
    try {
      await axios.delete(`${API_URL}/api/users/${userId}`, {
        headers: { Authorization: `Bearer ${token}` }
      });
      fetchUsers();
    } catch (error) {
      console.error('Error deleting user:', error);
    }
  };

  return (
    <div className="user-management">
      <h2>User Management</h2>
      <table>
        <thead>
          <tr>
            <th>Name</th>
            <th>Email</th>
            <th>Role</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {users.map(user => (
            <tr key={user.id}>
              <td>{user.name}</td>
              <td>{user.email}</td>
              <td>
                <select 
                  value={user.role} 
                  onChange={(e) => updateUserRole(user.id, e.target.value)}
                >
                  <option value="user">User</option>
                  <option value="admin">Admin</option>
                </select>
              </td>
              <td>
                <button onClick={() => deleteUser(user.id)}>Delete</button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

export default AdminUserManagement;
```

## Configuration

### MongoDB Connection

```javascript
// backend/config/db.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
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

```javascript
// backend/server.js
const cors = require('cors');
const express = require('express');

const app = express();

app.use(cors({
  origin: process.env.CLIENT_URL || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Connection Errors

```python
# ml-service/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Token Expiration Handling

```javascript
// frontend/utils/axios.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

api.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Database Connection Issues

Check MongoDB is running:

```bash
# Local MongoDB
sudo systemctl status mongod

# Or using Docker
docker ps | grep mongo
```

Verify connection string in `.env`:

```env
# Local
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt

# MongoDB Atlas
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/enterprise-user-mgmt
```

### ML Model Loading Errors

Ensure model files exist:

```bash
cd ml-service
mkdir -p models
# Add your trained models to this directory
```

Initialize models if missing:

```python
# ml-service/utils/init_models.py
import pickle
from sklearn.ensemble import RandomForestClassifier

def initialize_models():
    # Create default models if not exist
    models = {
        'ticket_classifier': RandomForestClassifier(),
        'risk_predictor': RandomForestClassifier(),
        'anomaly_detector': IsolationForest()
    }
    
    for name, model in models.items():
        with open(f'models/{name}.pkl', 'wb') as f:
            pickle.dump(model, f)
```
