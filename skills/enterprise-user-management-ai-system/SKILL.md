---
name: enterprise-user-management-ai-system
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, and intelligent ticket routing
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user management dashboard"
  - "create task tracking with AI analytics"
  - "build support ticket system with AI routing"
  - "integrate AI risk detection in user management"
  - "develop admin dashboard with user analytics"
  - "add AI burnout detection to task management"
  - "configure enterprise user management with ML service"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket systems with AI-powered insights. It provides risk detection, anomaly detection, burnout analysis, and predictive project analytics using machine learning models integrated through a FastAPI service.

**Key capabilities:**
- JWT-based authentication and role-based access control
- Kanban-style task management with time tracking
- AI-powered ticket classification and routing
- Real-time risk and anomaly detection
- Burnout prediction based on workload analysis
- Admin analytics dashboard with audit logs

## Architecture

The system consists of three main components:

1. **Frontend** (React.js) - User and admin dashboards
2. **Backend** (Node.js) - REST API, authentication, business logic
3. **ML Service** (FastAPI + scikit-learn) - AI analytics and predictions

## Installation

### Prerequisites

- Node.js 14+ and npm
- Python 3.8+
- MongoDB (local or Atlas)

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Setup backend
cd backend
npm install
cp .env.example .env
# Configure MongoDB connection and JWT secret in .env

# Setup ML service
cd ../ml-service
pip install -r requirements.txt
cp .env.example .env

# Setup frontend
cd ../frontend
npm install
cp .env.example .env
# Set REACT_APP_API_URL and REACT_APP_ML_API_URL
```

### Environment Configuration

**Backend (.env):**
```bash
MONGODB_URI=mongodb://localhost:27017/enterprise-ums
JWT_SECRET=your_jwt_secret_here
PORT=5000
ML_SERVICE_URL=http://localhost:8000
```

**ML Service (.env):**
```bash
MODEL_PATH=./models
PORT=8000
```

**Frontend (.env):**
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_API_URL=http://localhost:8000
```

## Running the Application

### Start Backend Server

```bash
cd backend
npm start
# Backend runs at http://localhost:5000
```

### Start ML Service

```bash
cd ml-service
uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Start Frontend

```bash
cd frontend
npm start
# Frontend runs at http://localhost:3000
```

## Backend API Usage

### Authentication

```javascript
// Register new user
const axios = require('axios');

async function registerUser() {
  const response = await axios.post('http://localhost:5000/api/auth/register', {
    username: 'john.doe',
    email: 'john@example.com',
    password: 'SecurePass123!',
    role: 'user' // or 'admin'
  });
  return response.data;
}

// Login
async function login() {
  const response = await axios.post('http://localhost:5000/api/auth/login', {
    email: 'john@example.com',
    password: 'SecurePass123!'
  });
  const { token, user } = response.data;
  // Store token for subsequent requests
  return token;
}

// Protected request with JWT
async function getUserProfile(token) {
  const response = await axios.get('http://localhost:5000/api/users/profile', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.data;
}
```

### User Management (Admin Only)

```javascript
// Fetch all users
async function getAllUsers(token) {
  const response = await axios.get('http://localhost:5000/api/admin/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.data;
}

// Create user
async function createUser(token, userData) {
  const response = await axios.post('http://localhost:5000/api/admin/users', {
    username: userData.username,
    email: userData.email,
    password: userData.password,
    role: userData.role,
    department: userData.department
  }, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.data;
}

// Update user
async function updateUser(token, userId, updates) {
  const response = await axios.put(
    `http://localhost:5000/api/admin/users/${userId}`,
    updates,
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}

// Delete user
async function deleteUser(token, userId) {
  await axios.delete(`http://localhost:5000/api/admin/users/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
}
```

### Task Management

```javascript
// Create task
async function createTask(token, taskData) {
  const response = await axios.post('http://localhost:5000/api/tasks', {
    title: taskData.title,
    description: taskData.description,
    assignedTo: taskData.userId,
    priority: 'high', // 'low', 'medium', 'high'
    status: 'todo', // 'todo', 'in-progress', 'done'
    dueDate: taskData.dueDate
  }, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.data;
}

// Get user tasks
async function getUserTasks(token) {
  const response = await axios.get('http://localhost:5000/api/tasks/my-tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.data;
}

// Update task status
async function updateTaskStatus(token, taskId, newStatus) {
  const response = await axios.patch(
    `http://localhost:5000/api/tasks/${taskId}/status`,
    { status: newStatus },
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}

// Track time on task
async function startTimeTracking(token, taskId) {
  const response = await axios.post(
    `http://localhost:5000/api/tasks/${taskId}/time-start`,
    {},
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}

async function stopTimeTracking(token, taskId) {
  const response = await axios.post(
    `http://localhost:5000/api/tasks/${taskId}/time-stop`,
    {},
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}
```

### Support Ticket Management

```javascript
// Create support ticket
async function createTicket(token, ticketData) {
  const response = await axios.post('http://localhost:5000/api/tickets', {
    title: ticketData.title,
    description: ticketData.description,
    priority: ticketData.priority,
    category: ticketData.category // 'technical', 'access', 'other'
  }, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.data;
}

// Get user tickets
async function getUserTickets(token) {
  const response = await axios.get('http://localhost:5000/api/tickets/my-tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.data;
}

// Update ticket (admin)
async function updateTicket(token, ticketId, updates) {
  const response = await axios.put(
    `http://localhost:5000/api/tickets/${ticketId}`,
    {
      status: updates.status, // 'open', 'in-progress', 'resolved', 'closed'
      assignedTo: updates.assignedTo,
      resolution: updates.resolution
    },
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.data;
}
```

## ML Service API Usage

### AI-Powered Analytics

```javascript
// Risk prediction for user
async function predictUserRisk(userId, behaviorData) {
  const response = await axios.post('http://localhost:8000/api/ml/risk-prediction', {
    userId: userId,
    loginFrequency: behaviorData.loginFrequency,
    failedLogins: behaviorData.failedLogins,
    taskCompletionRate: behaviorData.taskCompletionRate,
    averageResponseTime: behaviorData.averageResponseTime,
    accessPatterns: behaviorData.accessPatterns
  });
  
  const { riskScore, riskLevel, factors } = response.data;
  // riskLevel: 'low', 'medium', 'high'
  return response.data;
}

// Anomaly detection
async function detectAnomaly(activityData) {
  const response = await axios.post('http://localhost:8000/api/ml/anomaly-detection', {
    userId: activityData.userId,
    loginTime: activityData.loginTime,
    ipAddress: activityData.ipAddress,
    location: activityData.location,
    deviceInfo: activityData.deviceInfo,
    actionsPerformed: activityData.actionsPerformed
  });
  
  const { isAnomaly, anomalyScore, reason } = response.data;
  return response.data;
}

// Burnout prediction
async function predictBurnout(userId, workloadData) {
  const response = await axios.post('http://localhost:8000/api/ml/burnout-prediction', {
    userId: userId,
    tasksAssigned: workloadData.tasksAssigned,
    tasksCompleted: workloadData.tasksCompleted,
    averageWorkHours: workloadData.averageWorkHours,
    overtimeHours: workloadData.overtimeHours,
    missedDeadlines: workloadData.missedDeadlines,
    stressIndicators: workloadData.stressIndicators
  });
  
  const { burnoutRisk, recommendation } = response.data;
  // burnoutRisk: 'low', 'moderate', 'high'
  return response.data;
}

// AI ticket classification and routing
async function classifyTicket(ticketContent) {
  const response = await axios.post('http://localhost:8000/api/ml/classify-ticket', {
    title: ticketContent.title,
    description: ticketContent.description
  });
  
  const { category, priority, suggestedAssignee, confidence } = response.data;
  return response.data;
}

// Predictive project insights
async function getPredictiveInsights(projectData) {
  const response = await axios.post('http://localhost:8000/api/ml/project-insights', {
    projectId: projectData.projectId,
    currentProgress: projectData.currentProgress,
    tasksRemaining: projectData.tasksRemaining,
    teamVelocity: projectData.teamVelocity,
    deadline: projectData.deadline,
    historicalData: projectData.historicalData
  });
  
  const { delayProbability, estimatedCompletion, recommendations } = response.data;
  return response.data;
}
```

## Frontend Integration Patterns

### React Component for User Dashboard

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [burnoutAlert, setBurnoutAlert] = useState(null);
  const token = localStorage.getItem('authToken');

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    try {
      const headers = { Authorization: `Bearer ${token}` };
      
      // Fetch tasks
      const tasksRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/tasks/my-tasks`,
        { headers }
      );
      setTasks(tasksRes.data);

      // Fetch tickets
      const ticketsRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/tickets/my-tickets`,
        { headers }
      );
      setTickets(ticketsRes.data);

      // Check burnout risk
      const burnoutRes = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-prediction`,
        {
          userId: localStorage.getItem('userId'),
          tasksAssigned: tasksRes.data.length,
          tasksCompleted: tasksRes.data.filter(t => t.status === 'done').length,
          averageWorkHours: 8.5,
          overtimeHours: 5,
          missedDeadlines: 2,
          stressIndicators: ['frequent-overtime', 'weekend-work']
        }
      );
      
      if (burnoutRes.data.burnoutRisk === 'high') {
        setBurnoutAlert(burnoutRes.data);
      }
    } catch (error) {
      console.error('Error fetching user data:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await axios.patch(
        `${process.env.REACT_APP_API_URL}/tasks/${taskId}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      fetchUserData(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="user-dashboard">
      {burnoutAlert && (
        <div className="alert alert-warning">
          ⚠️ Burnout Risk Detected: {burnoutAlert.recommendation}
        </div>
      )}
      
      <div className="kanban-board">
        {['todo', 'in-progress', 'done'].map(status => (
          <div key={status} className="kanban-column">
            <h3>{status.toUpperCase()}</h3>
            {tasks.filter(t => t.status === status).map(task => (
              <TaskCard 
                key={task._id} 
                task={task} 
                onMove={moveTask}
              />
            ))}
          </div>
        ))}
      </div>

      <div className="tickets-section">
        <h2>Support Tickets</h2>
        {tickets.map(ticket => (
          <TicketCard key={ticket._id} ticket={ticket} />
        ))}
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [riskUsers, setRiskUsers] = useState([]);
  const [anomalies, setAnomalies] = useState([]);
  const token = localStorage.getItem('authToken');

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const headers = { Authorization: `Bearer ${token}` };
      
      // Fetch all users
      const usersRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/admin/users`,
        { headers }
      );
      setUsers(usersRes.data);

      // Run risk analysis for each user
      const riskAnalysis = await Promise.all(
        usersRes.data.map(async (user) => {
          const riskRes = await axios.post(
            `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`,
            {
              userId: user._id,
              loginFrequency: user.stats?.loginFrequency || 0,
              failedLogins: user.stats?.failedLogins || 0,
              taskCompletionRate: user.stats?.taskCompletionRate || 100,
              averageResponseTime: user.stats?.avgResponseTime || 0,
              accessPatterns: user.stats?.accessPatterns || []
            }
          );
          return { user, risk: riskRes.data };
        })
      );

      const highRisk = riskAnalysis.filter(
        item => item.risk.riskLevel === 'high'
      );
      setRiskUsers(highRisk);

      // Fetch recent anomalies
      const anomaliesRes = await axios.get(
        `${process.env.REACT_APP_API_URL}/admin/anomalies`,
        { headers }
      );
      setAnomalies(anomaliesRes.data);

    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  return (
    <div className="admin-analytics">
      <div className="stats-overview">
        <div className="stat-card">
          <h3>Total Users</h3>
          <p>{users.length}</p>
        </div>
        <div className="stat-card alert">
          <h3>High Risk Users</h3>
          <p>{riskUsers.length}</p>
        </div>
        <div className="stat-card warning">
          <h3>Anomalies Detected</h3>
          <p>{anomalies.length}</p>
        </div>
      </div>

      <div className="risk-users-section">
        <h2>High Risk Users</h2>
        {riskUsers.map(({ user, risk }) => (
          <div key={user._id} className="risk-card">
            <h4>{user.username}</h4>
            <p>Risk Score: {risk.riskScore.toFixed(2)}</p>
            <p>Factors: {risk.factors.join(', ')}</p>
          </div>
        ))}
      </div>

      <div className="anomalies-section">
        <h2>Recent Anomalies</h2>
        {anomalies.map(anomaly => (
          <div key={anomaly._id} className="anomaly-card">
            <p>{anomaly.userId} - {anomaly.reason}</p>
            <small>{new Date(anomaly.timestamp).toLocaleString()}</small>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Common Patterns

### Protected Route Implementation

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('authToken');
  const userRole = localStorage.getItem('userRole');

  if (!token) {
    return <Navigate to="/login" replace />;
  }

  if (adminOnly && userRole !== 'admin') {
    return <Navigate to="/dashboard" replace />;
  }

  return children;
};

export default ProtectedRoute;
```

### Axios Interceptor for Auth

```javascript
import axios from 'axios';

// Setup axios interceptor for authentication
axios.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

axios.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('authToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### Real-time Notification Handling

```javascript
import { useState, useEffect } from 'react';
import io from 'socket.io-client';

const useNotifications = () => {
  const [notifications, setNotifications] = useState([]);
  const token = localStorage.getItem('authToken');

  useEffect(() => {
    const socket = io(process.env.REACT_APP_API_URL, {
      auth: { token }
    });

    socket.on('notification', (notification) => {
      setNotifications(prev => [notification, ...prev]);
    });

    socket.on('ai-alert', (alert) => {
      // Handle AI-generated alerts (burnout, anomaly, risk)
      if (alert.type === 'burnout') {
        console.warn('Burnout alert:', alert.message);
      }
    });

    return () => socket.disconnect();
  }, [token]);

  return notifications;
};

export default useNotifications;
```

## Configuration

### MongoDB Schema Examples

**User Schema:**
```javascript
const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  stats: {
    loginFrequency: Number,
    failedLogins: Number,
    taskCompletionRate: Number,
    avgResponseTime: Number,
    accessPatterns: [String]
  },
  createdAt: { type: Date, default: Date.now }
});
```

**Task Schema:**
```javascript
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
  timeTracking: {
    startTime: Date,
    endTime: Date,
    totalTime: Number
  },
  dueDate: Date,
  createdAt: { type: Date, default: Date.now }
});
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Refresh token mechanism
async function refreshToken() {
  try {
    const refreshToken = localStorage.getItem('refreshToken');
    const response = await axios.post(
      `${process.env.REACT_APP_API_URL}/auth/refresh`,
      { refreshToken }
    );
    localStorage.setItem('authToken', response.data.token);
    return response.data.token;
  } catch (error) {
    localStorage.clear();
    window.location.href = '/login';
  }
}
```

### ML Service Connection Issues

```javascript
// Health check for ML service
async function checkMLServiceHealth() {
  try {
    const response = await axios.get(
      `${process.env.REACT_APP_ML_API_URL}/health`,
      { timeout: 5000 }
    );
    return response.data.status === 'healthy';
  } catch (error) {
    console.error('ML service unavailable:', error);
    return false;
  }
}

// Fallback when ML service is down
async function getTaskPriorityWithFallback(taskData) {
  const mlHealthy = await checkMLServiceHealth();
  
  if (mlHealthy) {
    try {
      const response = await axios.post(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`,
        taskData
      );
      return response.data.priority;
    } catch (error) {
      console.warn('ML classification failed, using fallback');
    }
  }
  
  // Fallback logic
  return taskData.title.toLowerCase().includes('urgent') ? 'high' : 'medium';
}
```

### CORS Configuration

```javascript
// Backend CORS setup (Express)
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Database Connection Issues

```javascript
// Robust MongoDB connection with retry
const mongoose = require('mongoose');

async function connectDB(retries = 5) {
  for (let i = 0; i < retries; i++) {
    try {
      await mongoose.connect(process.env.MONGODB_URI, {
        useNewUrlParser: true,
        useUnifiedTopology: true,
        serverSelectionTimeoutMS: 5000
      });
      console.log('MongoDB connected successfully');
      return;
    } catch (error) {
      console.error(`MongoDB connection attempt ${i + 1} failed:`, error);
      if (i === retries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 3000));
    }
  }
}

connectDB().catch(err => {
  console.error('Fatal: Could not connect to MongoDB', err);
  process.exit(1);
});
```

## Deployment Considerations

### Production Environment Variables

```bash
# Backend
NODE_ENV=production
MONGODB_URI=${MONGODB_ATLAS_URI}
JWT_SECRET=${SECURE_JWT_SECRET}
JWT_REFRESH_SECRET=${SECURE_REFRESH_SECRET}
ML_SERVICE_URL=${ML_SERVICE_PRODUCTION_URL}
FRONTEND_URL=${FRONTEND_PRODUCTION_URL}

# ML Service
MODEL_PATH=/app/models
WORKERS=4

# Frontend
REACT_APP_API_URL=${BACKEND_PRODUCTION_URL}/api
REACT_APP_ML_API_URL=${ML_SERVICE_PRODUCTION_URL}
```

### Performance Optimization

```javascript
// Implement caching for frequent ML predictions
const NodeCache = require('node-cache');
const mlCache = new NodeCache({ stdTTL: 300 }); // 5 minutes

async function getCachedRiskPrediction(userId, behaviorData) {
  const cacheKey = `risk_${userId}`;
  const cached = mlCache.get(cacheKey);
  
  if (cached) {
    return cached;
  }
  
  const prediction = await axios.post(
    `${process.env.ML_SERVICE_URL}/api/ml/risk-prediction`,
    { userId, ...behaviorData }
  );
  
  mlCache.set(cacheKey, prediction.data);
  return prediction.data;
}
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI coding agents to effectively assist developers in implementing, configuring, and troubleshooting the system.
