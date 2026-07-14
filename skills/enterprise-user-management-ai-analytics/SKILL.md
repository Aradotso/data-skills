---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create admin dashboard with user tracking"
  - "build kanban task board with time tracking"
  - "integrate AI ticket classification system"
  - "add anomaly detection to user management"
  - "setup JWT authentication for enterprise app"
  - "implement burnout detection for employees"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management platform that combines traditional CRUD operations with AI-powered analytics. It provides user/task management, Kanban boards, support tickets, and ML-driven insights including risk prediction, anomaly detection, burnout analysis, and predictive project delays.

## Project Architecture

The system consists of three main components:

1. **Frontend**: React.js application (port 3000)
2. **Backend**: Node.js REST API with MongoDB (port 5000)
3. **ML Service**: FastAPI service with scikit-learn and River (port 8000)

## Installation

### Clone and Setup All Services

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
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
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

Create `.env` file:

```bash
MODEL_PATH=./models
LOG_LEVEL=info
```

Start ML service:

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file:

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Backend API Usage

### Authentication

**Register User:**

```javascript
const axios = require('axios');

async function registerUser() {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/register', {
      name: 'John Doe',
      email: 'john@example.com',
      password: 'securePassword123',
      role: 'user' // or 'admin'
    });
    console.log('Token:', response.data.token);
    return response.data.token;
  } catch (error) {
    console.error('Registration failed:', error.response.data);
  }
}
```

**Login:**

```javascript
async function loginUser(email, password) {
  const response = await axios.post('http://localhost:5000/api/auth/login', {
    email,
    password
  });
  return response.data.token;
}
```

**Authenticated Requests:**

```javascript
const token = await loginUser('john@example.com', 'securePassword123');

const config = {
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  }
};

// Use this config for all authenticated requests
```

### User Management (Admin Only)

**Get All Users:**

```javascript
async function getAllUsers(token) {
  const response = await axios.get('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.data;
}
```

**Create User:**

```javascript
async function createUser(token, userData) {
  const response = await axios.post(
    'http://localhost:5000/api/users',
    {
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role,
      department: userData.department
    },
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}
```

**Update User:**

```javascript
async function updateUser(token, userId, updates) {
  const response = await axios.put(
    `http://localhost:5000/api/users/${userId}`,
    updates,
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}
```

**Delete User:**

```javascript
async function deleteUser(token, userId) {
  await axios.delete(`http://localhost:5000/api/users/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
}
```

### Task Management

**Create Task:**

```javascript
async function createTask(token, taskData) {
  const response = await axios.post(
    'http://localhost:5000/api/tasks',
    {
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: 'high', // low, medium, high
      status: 'todo', // todo, in-progress, done
      dueDate: taskData.dueDate
    },
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}
```

**Get User Tasks:**

```javascript
async function getUserTasks(token) {
  const response = await axios.get('http://localhost:5000/api/tasks/my-tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.data;
}
```

**Update Task Status:**

```javascript
async function updateTaskStatus(token, taskId, newStatus) {
  const response = await axios.patch(
    `http://localhost:5000/api/tasks/${taskId}/status`,
    { status: newStatus }, // 'todo', 'in-progress', 'done'
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}
```

**Track Time on Task:**

```javascript
async function logTime(token, taskId, timeSpent) {
  const response = await axios.post(
    `http://localhost:5000/api/tasks/${taskId}/time`,
    { 
      timeSpent: timeSpent, // in minutes
      date: new Date()
    },
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}
```

### Support Tickets

**Create Ticket:**

```javascript
async function createTicket(token, ticketData) {
  const response = await axios.post(
    'http://localhost:5000/api/tickets',
    {
      title: ticketData.title,
      description: ticketData.description,
      priority: 'medium', // low, medium, high, critical
      category: 'technical' // technical, access, general
    },
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}
```

**Get Tickets:**

```javascript
async function getMyTickets(token) {
  const response = await axios.get('http://localhost:5000/api/tickets/my-tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.data;
}
```

**Update Ticket:**

```javascript
async function updateTicket(token, ticketId, updates) {
  const response = await axios.put(
    `http://localhost:5000/api/tickets/${ticketId}`,
    {
      status: 'in-progress', // open, in-progress, resolved, closed
      assignedTo: updates.assignedTo,
      notes: updates.notes
    },
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}
```

## ML Service API Usage

### AI Ticket Classification

```javascript
async function classifyTicket(ticketText) {
  const response = await axios.post('http://localhost:8000/api/ml/classify-ticket', {
    text: ticketText,
    description: 'Unable to access dashboard after login'
  });
  
  console.log('Category:', response.data.category);
  console.log('Priority:', response.data.priority);
  console.log('Suggested Assignee:', response.data.suggestedAssignee);
  return response.data;
}
```

### Risk Prediction

```javascript
async function predictUserRisk(userId, userData) {
  const response = await axios.post('http://localhost:8000/api/ml/predict-risk', {
    userId: userId,
    loginFrequency: userData.loginFrequency,
    taskCompletionRate: userData.taskCompletionRate,
    averageResponseTime: userData.averageResponseTime,
    failedLogins: userData.failedLogins,
    lastActivityDays: userData.lastActivityDays
  });
  
  console.log('Risk Level:', response.data.riskLevel); // low, medium, high
  console.log('Risk Score:', response.data.riskScore);
  console.log('Factors:', response.data.factors);
  return response.data;
}
```

### Anomaly Detection

```javascript
async function detectAnomaly(activityData) {
  const response = await axios.post('http://localhost:8000/api/ml/detect-anomaly', {
    userId: activityData.userId,
    timestamp: new Date().toISOString(),
    loginTime: activityData.loginTime,
    ipAddress: activityData.ipAddress,
    location: activityData.location,
    deviceType: activityData.deviceType,
    actionCount: activityData.actionCount
  });
  
  console.log('Is Anomaly:', response.data.isAnomaly);
  console.log('Anomaly Score:', response.data.anomalyScore);
  console.log('Reason:', response.data.reason);
  return response.data;
}
```

### Burnout Detection

```javascript
async function detectBurnout(userId, workloadData) {
  const response = await axios.post('http://localhost:8000/api/ml/detect-burnout', {
    userId: userId,
    hoursWorked: workloadData.hoursWorked,
    tasksCompleted: workloadData.tasksCompleted,
    overdueTasksCount: workloadData.overdueTasksCount,
    weekendWork: workloadData.weekendWork,
    avgTaskDuration: workloadData.avgTaskDuration,
    stressLevel: workloadData.stressLevel // 1-10
  });
  
  console.log('Burnout Risk:', response.data.burnoutRisk); // low, moderate, high
  console.log('Recommendations:', response.data.recommendations);
  return response.data;
}
```

### Predictive Project Insights

```javascript
async function predictProjectDelay(projectData) {
  const response = await axios.post('http://localhost:8000/api/ml/predict-delay', {
    projectId: projectData.projectId,
    tasksTotal: projectData.tasksTotal,
    tasksCompleted: projectData.tasksCompleted,
    daysRemaining: projectData.daysRemaining,
    teamSize: projectData.teamSize,
    complexity: projectData.complexity, // 1-5
    dependenciesCount: projectData.dependenciesCount
  });
  
  console.log('Delay Probability:', response.data.delayProbability);
  console.log('Predicted Delay Days:', response.data.predictedDelayDays);
  console.log('Recommendations:', response.data.recommendations);
  return response.data;
}
```

## React Frontend Patterns

### Authentication Context

```javascript
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (token) {
      fetchUser();
    } else {
      setLoading(false);
    }
  }, [token]);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/api/auth/me`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      setUser(response.data);
    } catch (error) {
      localStorage.removeItem('token');
      setToken(null);
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
      email,
      password
    });
    localStorage.setItem('token', response.data.token);
    setToken(response.data.token);
    setUser(response.data.user);
  };

  const logout = () => {
    localStorage.removeItem('token');
    setToken(null);
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, token, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Kanban Board Component

```javascript
import React, { useState, useEffect, useContext } from 'react';
import axios from 'axios';
import { AuthContext } from '../context/AuthContext';

const KanbanBoard = () => {
  const { token } = useContext(AuthContext);
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const response = await axios.get(
      `${process.env.REACT_APP_API_URL}/api/tasks/my-tasks`,
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    
    const categorized = {
      todo: response.data.filter(t => t.status === 'todo'),
      inProgress: response.data.filter(t => t.status === 'in-progress'),
      done: response.data.filter(t => t.status === 'done')
    };
    setTasks(categorized);
  };

  const moveTask = async (taskId, newStatus) => {
    await axios.patch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
      { status: newStatus },
      { headers: { 'Authorization': `Bearer ${token}` } }
    );
    fetchTasks();
  };

  return (
    <div className="kanban-board">
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onMove={(id) => moveTask(id, 'in-progress')} 
          />
        ))}
      </div>
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
          <TaskCard 
            key={task._id} 
            task={task} 
            onMove={(id) => moveTask(id, 'done')} 
          />
        ))}
      </div>
      <div className="column">
        <h3>Done</h3>
        {tasks.done.map(task => (
          <TaskCard key={task._id} task={task} />
        ))}
      </div>
    </div>
  );
};
```

### AI Analytics Dashboard

```javascript
import React, { useState, useEffect, useContext } from 'react';
import axios from 'axios';
import { AuthContext } from '../context/AuthContext';

const AIAnalyticsDashboard = () => {
  const { token, user } = useContext(AuthContext);
  const [analytics, setAnalytics] = useState({
    riskLevel: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      // Get risk prediction
      const riskResponse = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/predict-risk`,
        {
          userId: user.id,
          loginFrequency: user.loginFrequency,
          taskCompletionRate: user.taskCompletionRate,
          averageResponseTime: user.avgResponseTime,
          failedLogins: user.failedLogins,
          lastActivityDays: user.lastActivityDays
        }
      );

      // Get burnout detection
      const burnoutResponse = await axios.post(
        `${process.env.REACT_APP_ML_URL}/api/ml/detect-burnout`,
        {
          userId: user.id,
          hoursWorked: user.weeklyHours,
          tasksCompleted: user.tasksCompleted,
          overdueTasksCount: user.overdueTasks,
          weekendWork: user.weekendHours,
          avgTaskDuration: user.avgTaskDuration,
          stressLevel: user.stressLevel
        }
      );

      setAnalytics({
        riskLevel: riskResponse.data,
        burnoutRisk: burnoutResponse.data,
        anomalies: user.recentAnomalies || []
      });
    } catch (error) {
      console.error('Failed to fetch analytics:', error);
    }
  };

  return (
    <div className="analytics-dashboard">
      <h2>AI-Powered Analytics</h2>
      
      <div className="analytics-card">
        <h3>Risk Level</h3>
        <p className={`risk-${analytics.riskLevel?.riskLevel}`}>
          {analytics.riskLevel?.riskLevel?.toUpperCase()}
        </p>
        <p>Score: {analytics.riskLevel?.riskScore}</p>
      </div>

      <div className="analytics-card">
        <h3>Burnout Risk</h3>
        <p className={`burnout-${analytics.burnoutRisk?.burnoutRisk}`}>
          {analytics.burnoutRisk?.burnoutRisk?.toUpperCase()}
        </p>
        <ul>
          {analytics.burnoutRisk?.recommendations?.map((rec, idx) => (
            <li key={idx}>{rec}</li>
          ))}
        </ul>
      </div>

      <div className="analytics-card">
        <h3>Recent Anomalies</h3>
        {analytics.anomalies.length === 0 ? (
          <p>No anomalies detected</p>
        ) : (
          <ul>
            {analytics.anomalies.map((anomaly, idx) => (
              <li key={idx}>
                {anomaly.timestamp}: {anomaly.reason}
              </li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};
```

## Configuration

### Backend Configuration (backend/.env)

```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt

# JWT
JWT_SECRET=your_secure_jwt_secret_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password

# Logging
LOG_LEVEL=info
```

### ML Service Configuration (ml-service/.env)

```bash
# Model Storage
MODEL_PATH=./models
RETRAIN_INTERVAL=86400

# Logging
LOG_LEVEL=info

# Performance
MAX_WORKERS=4
BATCH_SIZE=32
```

### Frontend Configuration (frontend/.env)

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENV=development
```

## Troubleshooting

### Backend Issues

**MongoDB Connection Failed:**

```javascript
// Ensure MongoDB is running
// Check connection string in .env

// Add connection retry logic
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
    setTimeout(connectDB, 5000); // Retry after 5 seconds
  }
};
```

**JWT Token Expired:**

```javascript
// Add token refresh logic
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      // Token expired, redirect to login
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### ML Service Issues

**Model Not Found:**

```bash
# Ensure models directory exists
mkdir -p ml-service/models

# Models are trained on first request, but you can pre-train:
# Create a script to initialize models
```

**Prediction Accuracy Low:**

```python
# Retrain models with more data
# In ml-service, add retraining endpoint

@app.post("/api/ml/retrain")
async def retrain_models():
    # Load historical data
    # Retrain all models
    # Save updated models
    return {"status": "Models retrained successfully"}
```

### Frontend Issues

**CORS Errors:**

```javascript
// In backend, ensure CORS is configured
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

**API Calls Failing:**

```javascript
// Add proper error handling
const apiCall = async (endpoint, options) => {
  try {
    const response = await axios(endpoint, options);
    return response.data;
  } catch (error) {
    if (error.response) {
      console.error('API Error:', error.response.data);
      throw new Error(error.response.data.message || 'API request failed');
    } else if (error.request) {
      throw new Error('No response from server');
    } else {
      throw new Error('Request setup failed');
    }
  }
};
```

## Common Patterns

### Protected Routes

```javascript
import React, { useContext } from 'react';
import { Navigate } from 'react-router-dom';
import { AuthContext } from '../context/AuthContext';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const { user, loading } = useContext(AuthContext);

  if (loading) return <div>Loading...</div>;
  
  if (!user) return <Navigate to="/login" />;
  
  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};
```

### Real-time Notifications

```javascript
import { useEffect, useContext } from 'react';
import { AuthContext } from '../context/AuthContext';

const useNotifications = () => {
  const { token } = useContext(AuthContext);

  useEffect(() => {
    const fetchNotifications = async () => {
      const response = await axios.get(
        `${process.env.REACT_APP_API_URL}/api/notifications`,
        { headers: { 'Authorization': `Bearer ${token}` } }
      );
      return response.data;
    };

    // Poll for notifications every 30 seconds
    const interval = setInterval(fetchNotifications, 30000);
    return () => clearInterval(interval);
  }, [token]);
};
```

### Audit Logging

```javascript
// Backend middleware for audit logging
const auditLog = async (req, res, next) => {
  const log = {
    userId: req.user?.id,
    action: `${req.method} ${req.path}`,
    timestamp: new Date(),
    ipAddress: req.ip,
    userAgent: req.get('user-agent')
  };
  
  await AuditLog.create(log);
  next();
};

// Apply to sensitive routes
app.use('/api/users', auditLog, userRoutes);
app.use('/api/admin', auditLog, adminRoutes);
```

This system provides a comprehensive enterprise user management solution with AI-powered insights for improved decision-making and operational efficiency.
