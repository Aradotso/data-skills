---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management with task tracking"
  - "build admin dashboard with AI insights"
  - "add risk detection and anomaly analysis"
  - "configure JWT authentication for user system"
  - "integrate ML service for burnout prediction"
  - "deploy user management with Kanban board"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack JavaScript application that combines traditional user management with AI-powered analytics. The system provides role-based access control, task tracking with Kanban boards, support ticket management, and ML-driven insights including risk detection, anomaly analysis, burnout prediction, and project delay forecasting.

## Installation

### Prerequisites

Ensure you have Node.js (v14+), Python (3.8+), and MongoDB installed.

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
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MODEL_PATH=./models
DATA_PATH=./data
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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Architecture Overview

The system consists of three main services:

1. **Backend (Node.js/Express)**: REST API for user management, authentication, tasks, and tickets
2. **ML Service (FastAPI/Python)**: AI analytics endpoints for predictions and insights
3. **Frontend (React)**: User interface with admin and user dashboards

## Backend API Usage

### Authentication

**User Registration**

```javascript
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
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

**User Login**

```javascript
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
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

**Get All Users**

```javascript
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

**Create User**

```javascript
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

**Update User**

```javascript
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

**Delete User**

```javascript
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

**Create Task**

```javascript
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
      status: 'todo' // 'todo', 'in-progress', 'done'
    })
  });
  return await response.json();
};
```

**Get User Tasks**

```javascript
const getUserTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks/my-tasks', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

**Update Task Status**

```javascript
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      status: newStatus // 'todo', 'in-progress', 'done'
    })
  });
  return await response.json();
};
```

**Track Time on Task**

```javascript
const trackTaskTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      timeSpent: timeSpent // in minutes
    })
  });
  return await response.json();
};
```

### Support Ticket Management

**Create Ticket**

```javascript
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
      category: ticketData.category, // 'technical', 'billing', 'general'
      priority: ticketData.priority
    })
  });
  return await response.json();
};
```

**Get All Tickets (Admin)**

```javascript
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return await response.json();
};
```

**Update Ticket Status**

```javascript
const updateTicketStatus = async (ticketId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      status: status // 'open', 'in-progress', 'resolved', 'closed'
    })
  });
  return await response.json();
};
```

## ML Service API Usage

### AI-Powered Ticket Classification

```javascript
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      text: ticketText
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', confidence: 0.85, suggestedDepartment: 'IT' }
  return result;
};
```

### Risk Prediction

```javascript
const predictUserRisk = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:8000/api/ml/risk-prediction/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  const result = await response.json();
  // Returns: { riskScore: 0.65, riskLevel: 'medium', factors: ['missed_deadlines', 'low_activity'] }
  return result;
};
```

### Anomaly Detection

```javascript
const detectAnomalies = async (userId, activityData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      userId: userId,
      loginTimes: activityData.loginTimes,
      activityPattern: activityData.activityPattern,
      dataAccess: activityData.dataAccess
    })
  });
  const result = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.78, alerts: ['unusual_login_time', 'excessive_data_access'] }
  return result;
};
```

### Burnout Detection

```javascript
const detectBurnout = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:8000/api/ml/burnout-detection/${userId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.82, recommendations: ['reduce_workload', 'schedule_break'] }
  return result;
};
```

### Project Delay Prediction

```javascript
const predictProjectDelay = async (projectId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:8000/api/ml/project-insights/${projectId}`, {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.72, estimatedDelay: 5, riskFactors: ['resource_shortage', 'scope_creep'] }
  return result;
};
```

## React Frontend Patterns

### Auth Context Setup

```javascript
// contexts/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));

  useEffect(() => {
    if (token) {
      fetchUserProfile();
    }
  }, [token]);

  const fetchUserProfile = async () => {
    try {
      const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/me`, {
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
      const data = await response.json();
      setUser(data);
    } catch (error) {
      console.error('Failed to fetch user profile', error);
      logout();
    }
  };

  const login = async (email, password) => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    setToken(data.token);
    setUser(data.user);
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
// components/KanbanBoard.js
import React, { useState, useEffect } from 'react';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/my-tasks`, {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    const categorized = {
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
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
      <Column title="To Do" tasks={tasks.todo} status="todo" moveTask={moveTask} />
      <Column title="In Progress" tasks={tasks.inProgress} status="in-progress" moveTask={moveTask} />
      <Column title="Done" tasks={tasks.done} status="done" moveTask={moveTask} />
    </div>
  );
};
```

### AI Insights Dashboard

```javascript
// components/AIInsightsDashboard.js
import React, { useState, useEffect } from 'react';

const AIInsightsDashboard = () => {
  const [insights, setInsights] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAIInsights();
  }, []);

  const fetchAIInsights = async () => {
    const token = localStorage.getItem('token');
    const user = JSON.parse(localStorage.getItem('user'));
    
    // Fetch risk prediction
    const riskResponse = await fetch(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction/${user.id}`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const riskData = await riskResponse.json();

    // Fetch burnout detection
    const burnoutResponse = await fetch(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection/${user.id}`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    const burnoutData = await burnoutResponse.json();

    setInsights({
      riskScore: riskData,
      burnoutRisk: burnoutData,
      anomalies: []
    });
  };

  return (
    <div className="ai-insights-dashboard">
      <div className="insight-card">
        <h3>Risk Score</h3>
        <p className={`risk-${insights.riskScore?.riskLevel}`}>
          {insights.riskScore?.riskScore?.toFixed(2) || 'N/A'}
        </p>
      </div>
      <div className="insight-card">
        <h3>Burnout Risk</h3>
        <p className={`burnout-${insights.burnoutRisk?.burnoutRisk}`}>
          {insights.burnoutRisk?.burnoutRisk || 'N/A'}
        </p>
      </div>
    </div>
  );
};
```

## Common Patterns

### Protected Routes

```javascript
// components/ProtectedRoute.js
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

### API Service Wrapper

```javascript
// services/api.js
const API_URL = process.env.REACT_APP_API_URL;
const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const getAuthHeaders = () => ({
  'Authorization': `Bearer ${localStorage.getItem('token')}`,
  'Content-Type': 'application/json'
});

export const api = {
  // User operations
  getUsers: () => 
    fetch(`${API_URL}/api/users`, { headers: getAuthHeaders() }).then(r => r.json()),
  
  createUser: (userData) =>
    fetch(`${API_URL}/api/users`, {
      method: 'POST',
      headers: getAuthHeaders(),
      body: JSON.stringify(userData)
    }).then(r => r.json()),

  // Task operations
  getTasks: () =>
    fetch(`${API_URL}/api/tasks/my-tasks`, { headers: getAuthHeaders() }).then(r => r.json()),

  createTask: (taskData) =>
    fetch(`${API_URL}/api/tasks`, {
      method: 'POST',
      headers: getAuthHeaders(),
      body: JSON.stringify(taskData)
    }).then(r => r.json()),

  // ML operations
  classifyTicket: (text) =>
    fetch(`${ML_API_URL}/api/ml/classify-ticket`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text })
    }).then(r => r.json()),

  getRiskPrediction: (userId) =>
    fetch(`${ML_API_URL}/api/ml/risk-prediction/${userId}`, {
      headers: getAuthHeaders()
    }).then(r => r.json())
};
```

## Configuration

### Backend Configuration (Node.js)

Key environment variables:

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
NODE_ENV=development
EMAIL_SERVICE=smtp
EMAIL_HOST=smtp.example.com
EMAIL_PORT=587
EMAIL_USER=your_email_user
EMAIL_PASS=your_email_password
```

### ML Service Configuration (Python)

Key environment variables:

```env
MODEL_PATH=./models
DATA_PATH=./data
LOG_LEVEL=INFO
ANOMALY_THRESHOLD=0.75
RISK_THRESHOLD=0.60
BURNOUT_THRESHOLD=0.70
```

### Frontend Configuration (React)

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_AI_FEATURES=true
```

## Troubleshooting

### JWT Token Expiration

If authentication fails unexpectedly:

```javascript
// Add token refresh logic
const refreshToken = async () => {
  try {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/refresh`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data.token;
  } catch (error) {
    // Redirect to login if refresh fails
    window.location.href = '/login';
  }
};
```

### CORS Issues

Ensure backend has proper CORS configuration:

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
}));
```

### ML Service Connection Errors

Check if ML service is running and accessible:

```javascript
const healthCheck = async () => {
  try {
    const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/health`);
    const data = await response.json();
    console.log('ML Service Status:', data.status);
  } catch (error) {
    console.error('ML Service unavailable:', error);
  }
};
```

### MongoDB Connection Issues

Verify MongoDB is running and connection string is correct:

```bash
# Check MongoDB status
sudo systemctl status mongod

# Test connection
mongosh mongodb://localhost:27017/enterprise-user-mgmt
```

### Task Status Update Failures

Ensure valid status transitions:

```javascript
const VALID_STATUSES = ['todo', 'in-progress', 'done'];

const updateTaskStatus = async (taskId, newStatus) => {
  if (!VALID_STATUSES.includes(newStatus)) {
    throw new Error(`Invalid status: ${newStatus}`);
  }
  
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message || 'Failed to update task status');
  }
  
  return await response.json();
};
```
