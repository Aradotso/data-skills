---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for risk detection, task management, and organizational insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create task tracking with burnout detection"
  - "build admin dashboard with AI insights"
  - "integrate ML service for user analytics"
  - "configure user management with kanban board"
  - "deploy enterprise management system with AI"
  - "add risk prediction to user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack application combining user/task management with AI-powered analytics. It provides role-based access control, Kanban task boards, support ticket management, and ML-driven insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Stack:**
- Frontend: React.js
- Backend: Node.js with Express (REST API)
- ML Service: FastAPI with scikit-learn and River
- Database: MongoDB
- Authentication: JWT

## Project Structure

```
Enterprise-User-Management-System-with-AI-Analytics/
├── frontend/          # React application
├── backend/           # Node.js API server
└── ml-service/        # FastAPI ML endpoints
```

## Installation & Setup

### 1. Clone and Setup Backend

```bash
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics/backend
npm install
```

Create `.env` file in `backend/`:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
NODE_ENV=development
```

Start backend:

```bash
npm start
```

Backend runs at `http://localhost:5000`

### 2. Setup ML Service

```bash
cd ../ml-service
pip install -r requirements.txt
```

Create `.env` file in `ml-service/`:

```env
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
API_PORT=8000
```

Start ML service:

```bash
uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

### 3. Setup Frontend

```bash
cd ../frontend
npm install
```

Create `.env` file in `frontend/`:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

Frontend runs at `http://localhost:3000`

## Backend API Structure

### Authentication Endpoints

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
  return response.json();
};

// POST /api/auth/login
const loginUser = async (email, password) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store JWT token
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management Endpoints

```javascript
// GET /api/users - Get all users (Admin only)
const getAllUsers = async (token) => {
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id - Delete user (Admin only)
const deleteUser = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management Endpoints

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
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
      status: 'todo', // todo, in-progress, done
      priority: taskData.priority, // low, medium, high
      deadline: taskData.deadline
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status }) // 'todo', 'in-progress', 'done'
  });
  return response.json();
};
```

### Support Ticket Endpoints

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category, // technical, billing, general
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets - Get all tickets (with filters)
const getTickets = async (filters, token) => {
  const queryParams = new URLSearchParams(filters).toString();
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tickets/:id - Update ticket
const updateTicket = async (ticketId, updates, token) => {
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};
```

## ML Service API

### Risk Prediction

```javascript
// POST /api/ml/risk-prediction
const predictUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      task_count: 15,
      overdue_tasks: 3,
      avg_completion_time: 48, // hours
      login_frequency: 0.8,
      ticket_count: 2
    })
  });
  return response.json();
  // Returns: { risk_score: 0.65, risk_level: "medium", factors: [...] }
};
```

### Anomaly Detection

```javascript
// POST /api/ml/anomaly-detection
const detectAnomalies = async (userActivity) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userActivity.userId,
      login_time: userActivity.loginTime, // timestamp
      login_location: userActivity.location,
      actions_per_hour: userActivity.actionsPerHour,
      data_accessed: userActivity.dataVolume
    })
  });
  return response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.85, details: {...} }
};
```

### Burnout Detection

```javascript
// POST /api/ml/burnout-detection
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      weekly_hours: 55,
      tasks_completed: 12,
      tasks_pending: 8,
      missed_deadlines: 3,
      avg_task_time: 6.5 // hours
    })
  });
  return response.json();
  // Returns: { burnout_risk: "high", score: 0.78, recommendations: [...] }
};
```

### AI Ticket Classification

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
  return response.json();
  // Returns: { category: "technical", priority: "high", confidence: 0.92 }
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/project-insights
const getProjectInsights = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/project-insights', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      team_size: projectData.teamSize,
      deadline: projectData.deadline,
      current_velocity: projectData.velocity
    })
  });
  return response.json();
  // Returns: { completion_probability: 0.72, estimated_delay: 5, bottlenecks: [...] }
};
```

## React Component Patterns

### Authentication Context

```javascript
// frontend/src/contexts/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      fetchCurrentUser();
    }
  }, [token]);

  const fetchCurrentUser = async () => {
    try {
      const response = await fetch('http://localhost:5000/api/auth/me', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await response.json();
      setUser(data.user);
    } catch (error) {
      console.error('Auth error:', error);
      logout();
    }
  };

  const login = async (email, password) => {
    const response = await fetch('http://localhost:5000/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    if (data.token) {
      localStorage.setItem('token', data.token);
      setToken(data.token);
      setUser(data.user);
    }
    return data;
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
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
// frontend/src/components/KanbanBoard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../contexts/AuthContext';

const KanbanBoard = () => {
  const { token, user } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], 'in-progress': [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await fetch(`http://localhost:5000/api/tasks/user/${user.id}`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const grouped = {
      todo: data.filter(t => t.status === 'todo'),
      'in-progress': data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
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
      {['todo', 'in-progress', 'done'].map(status => (
        <div key={status} className="kanban-column">
          <h3>{status.toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div key={task.id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <select 
                value={task.status} 
                onChange={(e) => moveTask(task.id, e.target.value)}
              >
                <option value="todo">To Do</option>
                <option value="in-progress">In Progress</option>
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

### Admin Dashboard with AI Insights

```javascript
// frontend/src/components/AdminDashboard.js
import React, { useState, useEffect, useContext } from 'react';
import { AuthContext } from '../contexts/AuthContext';

const AdminDashboard = () => {
  const { token } = useContext(AuthContext);
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);
  const [riskUsers, setRiskUsers] = useState([]);

  useEffect(() => {
    fetchDashboardData();
  }, []);

  const fetchDashboardData = async () => {
    // Fetch users
    const usersRes = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const usersData = await usersRes.json();
    setUsers(usersData);

    // Fetch analytics
    const analyticsRes = await fetch('http://localhost:5000/api/analytics', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const analyticsData = await analyticsRes.json();
    setAnalytics(analyticsData);

    // Fetch AI risk predictions for all users
    const riskPromises = usersData.map(user =>
      fetch('http://localhost:8000/api/ml/risk-prediction', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: user.id })
      }).then(r => r.json())
    );
    const riskResults = await Promise.all(riskPromises);
    const highRisk = riskResults
      .filter(r => r.risk_level === 'high')
      .map((r, i) => ({ ...usersData[i], riskScore: r.risk_score }));
    setRiskUsers(highRisk);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>
      
      <div className="analytics-overview">
        <div className="metric">
          <h3>Total Users</h3>
          <p>{analytics?.totalUsers || 0}</p>
        </div>
        <div className="metric">
          <h3>Active Tasks</h3>
          <p>{analytics?.activeTasks || 0}</p>
        </div>
        <div className="metric">
          <h3>Open Tickets</h3>
          <p>{analytics?.openTickets || 0}</p>
        </div>
      </div>

      <div className="risk-alerts">
        <h2>High Risk Users (AI Detected)</h2>
        {riskUsers.map(user => (
          <div key={user.id} className="alert-card">
            <p><strong>{user.name}</strong> - Risk Score: {user.riskScore.toFixed(2)}</p>
            <button onClick={() => viewUserDetails(user.id)}>View Details</button>
          </div>
        ))}
      </div>

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
                <td>{user.role}</td>
                <td>
                  <button onClick={() => editUser(user.id)}>Edit</button>
                  <button onClick={() => deleteUser(user.id)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  );
};

export default AdminDashboard;
```

## Common Patterns

### Protected Routes

```javascript
// frontend/src/components/ProtectedRoute.js
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../contexts/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, token } = useContext(AuthContext);

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user?.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Real-time AI Insights Hook

```javascript
// frontend/src/hooks/useAIInsights.js
import { useState, useEffect } from 'react';

export const useAIInsights = (userId) => {
  const [insights, setInsights] = useState({
    risk: null,
    burnout: null,
    anomalies: []
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchInsights = async () => {
      try {
        // Parallel fetch all AI insights
        const [riskRes, burnoutRes] = await Promise.all([
          fetch('http://localhost:8000/api/ml/risk-prediction', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ user_id: userId })
          }),
          fetch('http://localhost:8000/api/ml/burnout-detection', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ user_id: userId })
          })
        ]);

        const [riskData, burnoutData] = await Promise.all([
          riskRes.json(),
          burnoutRes.json()
        ]);

        setInsights({
          risk: riskData,
          burnout: burnoutData,
          anomalies: []
        });
      } catch (error) {
        console.error('Error fetching AI insights:', error);
      } finally {
        setLoading(false);
      }
    };

    if (userId) {
      fetchInsights();
      // Refresh insights every 5 minutes
      const interval = setInterval(fetchInsights, 5 * 60 * 1000);
      return () => clearInterval(interval);
    }
  }, [userId]);

  return { insights, loading };
};
```

## Configuration

### Backend Configuration (backend/config/config.js)

```javascript
module.exports = {
  mongodb: {
    uri: process.env.MONGODB_URI || 'mongodb://localhost:27017/enterprise_user_mgmt',
    options: {
      useNewUrlParser: true,
      useUnifiedTopology: true
    }
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRE || '7d'
  },
  ml: {
    apiUrl: process.env.ML_API_URL || 'http://localhost:8000',
    timeout: 10000
  },
  server: {
    port: process.env.PORT || 5000,
    corsOrigin: process.env.CORS_ORIGIN || 'http://localhost:3000'
  }
};
```

### ML Service Configuration (ml-service/config.py)

```python
import os
from pydantic import BaseSettings

class Settings(BaseSettings):
    mongodb_uri: str = os.getenv("MONGODB_URI", "mongodb://localhost:27017/enterprise_user_mgmt")
    model_path: str = os.getenv("MODEL_PATH", "./models")
    api_port: int = int(os.getenv("API_PORT", 8000))
    
    # ML Model Parameters
    risk_threshold: float = 0.7
    anomaly_threshold: float = 0.8
    burnout_threshold: float = 0.75
    
    class Config:
        env_file = ".env"

settings = Settings()
```

## Troubleshooting

### JWT Token Issues

```javascript
// Check token validity
const verifyToken = async (token) => {
  try {
    const response = await fetch('http://localhost:5000/api/auth/verify', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    return response.ok;
  } catch (error) {
    console.error('Token verification failed:', error);
    return false;
  }
};

// Refresh token if expired
const refreshAuthToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch('http://localhost:5000/api/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};
```

### MongoDB Connection Issues

```javascript
// backend/utils/db.js
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
    console.error('MongoDB connection error:', error.message);
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

module.exports = connectDB;
```

### ML Model Loading Issues

```python
# ml-service/utils/model_loader.py
import os
import pickle
from typing import Optional

def load_model(model_name: str) -> Optional[object]:
    """Load ML model with error handling"""
    model_path = os.path.join(settings.model_path, f"{model_name}.pkl")
    
    try:
        if not os.path.exists(model_path):
            print(f"Model {model_name} not found, using default")
            return None
        
        with open(model_path, 'rb') as f:
            model = pickle.load(f)
        print(f"Model {model_name} loaded successfully")
        return model
    except Exception as e:
        print(f"Error loading model {model_name}: {str(e)}")
        return None

def save_model(model: object, model_name: str):
    """Save trained model"""
    os.makedirs(settings.model_path, exist_ok=True)
    model_path = os.path.join(settings.model_path, f"{model_name}.pkl")
    
    with open(model_path, 'wb') as f:
        pickle.dump(model, f)
    print(f"Model {model_name} saved to {model_path}")
```

### CORS Configuration

```javascript
// backend/middleware/cors.js
const cors = require('cors');

const corsOptions = {
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  optionsSuccessStatus: 200,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
};

module.exports = cors(corsOptions);
```

### API Error Handling

```javascript
// frontend/src/utils/api.js
export const handleApiError = (error) => {
  if (error.response) {
    // Server responded with error status
    switch (error.response.status) {
      case 401:
        localStorage.removeItem('token');
        window.location.href = '/login';
        break;
      case 403:
        alert('You do not have permission to perform this action');
        break;
      case 404:
        alert('Resource not found');
        break;
      case 500:
        alert('Server error. Please try again later');
        break;
      default:
        alert(error.response.data.message || 'An error occurred');
    }
  } else if (error.request) {
    // Request made but no response
    alert('Network error. Please check your connection');
  } else {
    alert('An unexpected error occurred');
  }
  console.error('API Error:', error);
};
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, including setup, API usage, React patterns, and troubleshooting for AI coding agents to effectively assist developers.
