---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, task management, and burnout analysis
triggers:
  - "set up enterprise user management with AI analytics"
  - "create user management system with AI insights"
  - "implement task tracking with burnout detection"
  - "build admin dashboard with anomaly detection"
  - "add AI-powered ticket classification system"
  - "configure user management with predictive analytics"
  - "deploy enterprise user system with ML service"
  - "integrate AI analytics into user management"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management platform that combines traditional user/task management with AI-powered analytics including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## What This Project Does

This system provides:
- **User Management**: Role-based access control, authentication via JWT, user CRUD operations
- **Task Management**: Kanban boards, time tracking, progress monitoring
- **Support System**: Ticket creation, tracking, and AI-powered classification
- **AI Analytics**: Risk detection, anomaly detection, burnout prediction, project delay forecasting
- **Admin Dashboard**: Organization-wide analytics, audit logs, alerts

The architecture consists of three services:
1. **Frontend** (React.js) - User interface
2. **Backend** (Node.js) - REST API and business logic
3. **ML Service** (FastAPI + scikit-learn) - AI/ML predictions

## Installation

### Prerequisites
- Node.js 14+
- Python 3.8+
- MongoDB

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
MONGODB_URI=mongodb://localhost:27017/enterprise_users
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
ML_SERVICE_PORT=8000
MODEL_PATH=./models
LOG_LEVEL=info
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

## Backend API Usage

### Authentication

```javascript
// User Registration
const axios = require('axios');

const registerUser = async () => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/register', {
      name: 'John Doe',
      email: 'john@example.com',
      password: 'securePassword123',
      role: 'user'
    });
    console.log('User registered:', response.data);
  } catch (error) {
    console.error('Registration failed:', error.response.data);
  }
};

// User Login
const loginUser = async () => {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
      email: 'john@example.com',
      password: 'securePassword123'
    });
    const token = response.data.token;
    // Store token for subsequent requests
    localStorage.setItem('authToken', token);
    return token;
  } catch (error) {
    console.error('Login failed:', error.response.data);
  }
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async (token) => {
  try {
    const response = await axios.get('http://localhost:5000/api/users', {
      headers: { Authorization: `Bearer ${token}` }
    });
    return response.data;
  } catch (error) {
    console.error('Error fetching users:', error.response.data);
  }
};

// Create user
const createUser = async (token, userData) => {
  try {
    const response = await axios.post('http://localhost:5000/api/users', 
      {
        name: userData.name,
        email: userData.email,
        role: userData.role,
        department: userData.department
      },
      {
        headers: { Authorization: `Bearer ${token}` }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Error creating user:', error.response.data);
  }
};

// Update user
const updateUser = async (token, userId, updates) => {
  try {
    const response = await axios.put(`http://localhost:5000/api/users/${userId}`, 
      updates,
      {
        headers: { Authorization: `Bearer ${token}` }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Error updating user:', error.response.data);
  }
};

// Delete user
const deleteUser = async (token, userId) => {
  try {
    await axios.delete(`http://localhost:5000/api/users/${userId}`, {
      headers: { Authorization: `Bearer ${token}` }
    });
  } catch (error) {
    console.error('Error deleting user:', error.response.data);
  }
};
```

### Task Management

```javascript
// Create task
const createTask = async (token, taskData) => {
  try {
    const response = await axios.post('http://localhost:5000/api/tasks',
      {
        title: taskData.title,
        description: taskData.description,
        assignedTo: taskData.userId,
        priority: taskData.priority, // 'low', 'medium', 'high'
        deadline: taskData.deadline,
        status: 'todo' // 'todo', 'in-progress', 'done'
      },
      {
        headers: { Authorization: `Bearer ${token}` }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Error creating task:', error.response.data);
  }
};

// Get user tasks
const getUserTasks = async (token, userId) => {
  try {
    const response = await axios.get(`http://localhost:5000/api/tasks/user/${userId}`, {
      headers: { Authorization: `Bearer ${token}` }
    });
    return response.data;
  } catch (error) {
    console.error('Error fetching tasks:', error.response.data);
  }
};

// Update task status
const updateTaskStatus = async (token, taskId, newStatus) => {
  try {
    const response = await axios.patch(`http://localhost:5000/api/tasks/${taskId}/status`,
      { status: newStatus },
      {
        headers: { Authorization: `Bearer ${token}` }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Error updating task status:', error.response.data);
  }
};

// Track time on task
const trackTime = async (token, taskId, timeSpent) => {
  try {
    const response = await axios.post(`http://localhost:5000/api/tasks/${taskId}/time`,
      { timeSpent }, // in minutes
      {
        headers: { Authorization: `Bearer ${token}` }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Error tracking time:', error.response.data);
  }
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (token, ticketData) => {
  try {
    const response = await axios.post('http://localhost:5000/api/tickets',
      {
        subject: ticketData.subject,
        description: ticketData.description,
        priority: ticketData.priority,
        category: ticketData.category
      },
      {
        headers: { Authorization: `Bearer ${token}` }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Error creating ticket:', error.response.data);
  }
};

// Get tickets
const getTickets = async (token, filters = {}) => {
  try {
    const response = await axios.get('http://localhost:5000/api/tickets', {
      headers: { Authorization: `Bearer ${token}` },
      params: filters // { status: 'open', priority: 'high' }
    });
    return response.data;
  } catch (error) {
    console.error('Error fetching tickets:', error.response.data);
  }
};
```

## ML Service API Usage

### Risk Prediction

```javascript
// Predict user risk score
const predictRisk = async (userId, userData) => {
  try {
    const response = await axios.post('http://localhost:8000/api/ml/risk-prediction', {
      user_id: userId,
      login_frequency: userData.loginFrequency,
      failed_login_attempts: userData.failedLogins,
      access_pattern: userData.accessPattern,
      data_access_volume: userData.dataVolume,
      off_hours_activity: userData.offHoursActivity
    });
    
    // Response: { risk_score: 0.75, risk_level: 'high', factors: [...] }
    return response.data;
  } catch (error) {
    console.error('Error predicting risk:', error.response.data);
  }
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomaly = async (userId, activityData) => {
  try {
    const response = await axios.post('http://localhost:8000/api/ml/anomaly-detection', {
      user_id: userId,
      timestamp: new Date().toISOString(),
      login_time: activityData.loginTime,
      location: activityData.location,
      device: activityData.device,
      actions_count: activityData.actionsCount,
      resources_accessed: activityData.resourcesAccessed
    });
    
    // Response: { is_anomaly: true, anomaly_score: 0.89, alert_level: 'critical' }
    return response.data;
  } catch (error) {
    console.error('Error detecting anomaly:', error.response.data);
  }
};
```

### Burnout Detection

```javascript
// Analyze user burnout risk
const detectBurnout = async (userId, workloadData) => {
  try {
    const response = await axios.post('http://localhost:8000/api/ml/burnout-detection', {
      user_id: userId,
      tasks_count: workloadData.tasksCount,
      average_hours_per_day: workloadData.avgHoursPerDay,
      overtime_hours: workloadData.overtimeHours,
      missed_deadlines: workloadData.missedDeadlines,
      task_completion_rate: workloadData.completionRate,
      stress_indicators: workloadData.stressIndicators
    });
    
    // Response: { burnout_risk: 'high', score: 0.82, recommendations: [...] }
    return response.data;
  } catch (error) {
    console.error('Error detecting burnout:', error.response.data);
  }
};
```

### Ticket Classification

```javascript
// Classify support ticket using AI
const classifyTicket = async (ticketData) => {
  try {
    const response = await axios.post('http://localhost:8000/api/ml/classify-ticket', {
      subject: ticketData.subject,
      description: ticketData.description,
      user_history: ticketData.userHistory
    });
    
    // Response: { category: 'technical', priority: 'high', suggested_assignee: 'dept_id' }
    return response.data;
  } catch (error) {
    console.error('Error classifying ticket:', error.response.data);
  }
};
```

### Project Delay Prediction

```javascript
// Predict project completion delays
const predictProjectDelay = async (projectData) => {
  try {
    const response = await axios.post('http://localhost:8000/api/ml/project-prediction', {
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      team_size: projectData.teamSize,
      average_task_completion_time: projectData.avgCompletionTime,
      dependencies_count: projectData.dependenciesCount,
      current_progress: projectData.progress
    });
    
    // Response: { delay_probability: 0.65, estimated_delay_days: 5, risk_factors: [...] }
    return response.data;
  } catch (error) {
    console.error('Error predicting delay:', error.response.data);
  }
};
```

## Frontend Integration Patterns

### React Hook for API Calls

```javascript
import { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

// Custom hook for fetching data
const useApi = (endpoint, dependencies = []) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const token = localStorage.getItem('authToken');
        const response = await axios.get(`${API_URL}${endpoint}`, {
          headers: { Authorization: `Bearer ${token}` }
        });
        setData(response.data);
        setLoading(false);
      } catch (err) {
        setError(err.message);
        setLoading(false);
      }
    };

    fetchData();
  }, dependencies);

  return { data, loading, error };
};

// Usage in component
const UserDashboard = () => {
  const userId = localStorage.getItem('userId');
  const { data: tasks, loading, error } = useApi(`/api/tasks/user/${userId}`, [userId]);

  if (loading) return <div>Loading tasks...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div className="dashboard">
      <h2>My Tasks</h2>
      {tasks && tasks.map(task => (
        <div key={task.id} className="task-card">
          <h3>{task.title}</h3>
          <p>{task.description}</p>
          <span className={`status ${task.status}`}>{task.status}</span>
        </div>
      ))}
    </div>
  );
};
```

### Kanban Board Component

```javascript
import React, { useState } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL;

const KanbanBoard = ({ tasks, onTaskUpdate }) => {
  const [draggedTask, setDraggedTask] = useState(null);

  const columns = {
    todo: tasks.filter(t => t.status === 'todo'),
    'in-progress': tasks.filter(t => t.status === 'in-progress'),
    done: tasks.filter(t => t.status === 'done')
  };

  const handleDragStart = (task) => {
    setDraggedTask(task);
  };

  const handleDrop = async (newStatus) => {
    if (!draggedTask) return;

    try {
      const token = localStorage.getItem('authToken');
      await axios.patch(
        `${API_URL}/api/tasks/${draggedTask.id}/status`,
        { status: newStatus },
        { headers: { Authorization: `Bearer ${token}` } }
      );
      onTaskUpdate();
      setDraggedTask(null);
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="kanban-board">
      {Object.entries(columns).map(([status, statusTasks]) => (
        <div
          key={status}
          className="kanban-column"
          onDragOver={(e) => e.preventDefault()}
          onDrop={() => handleDrop(status)}
        >
          <h3>{status.toUpperCase()}</h3>
          {statusTasks.map(task => (
            <div
              key={task.id}
              className="task-card"
              draggable
              onDragStart={() => handleDragStart(task)}
            >
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <span className={`priority ${task.priority}`}>{task.priority}</span>
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
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const ML_API_URL = process.env.REACT_APP_ML_API_URL;

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    const fetchAnalytics = async () => {
      try {
        // Fetch user data from backend
        const token = localStorage.getItem('authToken');
        const userDataResponse = await axios.get(
          `${process.env.REACT_APP_API_URL}/api/users/${userId}/analytics`,
          { headers: { Authorization: `Bearer ${token}` } }
        );
        
        const userData = userDataResponse.data;

        // Get AI predictions
        const [riskPrediction, burnoutPrediction] = await Promise.all([
          axios.post(`${ML_API_URL}/api/ml/risk-prediction`, {
            user_id: userId,
            login_frequency: userData.loginFrequency,
            failed_login_attempts: userData.failedLogins,
            access_pattern: userData.accessPattern,
            data_access_volume: userData.dataVolume,
            off_hours_activity: userData.offHoursActivity
          }),
          axios.post(`${ML_API_URL}/api/ml/burnout-detection`, {
            user_id: userId,
            tasks_count: userData.tasksCount,
            average_hours_per_day: userData.avgHoursPerDay,
            overtime_hours: userData.overtimeHours,
            missed_deadlines: userData.missedDeadlines,
            task_completion_rate: userData.completionRate,
            stress_indicators: userData.stressIndicators
          })
        ]);

        setAnalytics({
          riskScore: riskPrediction.data,
          burnoutRisk: burnoutPrediction.data,
          anomalies: userData.recentAnomalies || []
        });
      } catch (error) {
        console.error('Error fetching analytics:', error);
      }
    };

    fetchAnalytics();
  }, [userId]);

  return (
    <div className="ai-analytics-dashboard">
      <h2>AI Analytics</h2>
      
      {analytics.riskScore && (
        <div className={`analytics-card risk-${analytics.riskScore.risk_level}`}>
          <h3>Security Risk Score</h3>
          <div className="score">{(analytics.riskScore.risk_score * 100).toFixed(0)}%</div>
          <p className="level">{analytics.riskScore.risk_level.toUpperCase()}</p>
          <ul>
            {analytics.riskScore.factors?.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
        </div>
      )}

      {analytics.burnoutRisk && (
        <div className={`analytics-card burnout-${analytics.burnoutRisk.burnout_risk}`}>
          <h3>Burnout Risk</h3>
          <div className="score">{(analytics.burnoutRisk.score * 100).toFixed(0)}%</div>
          <p className="level">{analytics.burnoutRisk.burnout_risk.toUpperCase()}</p>
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {analytics.burnoutRisk.recommendations?.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
      )}

      {analytics.anomalies.length > 0 && (
        <div className="analytics-card anomalies">
          <h3>Recent Anomalies Detected</h3>
          <ul>
            {analytics.anomalies.map((anomaly, idx) => (
              <li key={idx}>
                <span className={`alert-${anomaly.alert_level}`}>
                  {anomaly.timestamp}: {anomaly.description}
                </span>
              </li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Configuration

### Backend Environment Variables

```bash
# Server
PORT=5000
NODE_ENV=production

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_users
DB_NAME=enterprise_users

# Authentication
JWT_SECRET=your_secure_jwt_secret_key_here
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional, for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@example.com
SMTP_PASS=your_email_password

# Logging
LOG_LEVEL=info
```

### ML Service Environment Variables

```bash
# Server
ML_SERVICE_PORT=8000
HOST=0.0.0.0

# Models
MODEL_PATH=./models
RETRAIN_INTERVAL=86400  # seconds (24 hours)

# Database (for model persistence)
MONGODB_URI=mongodb://localhost:27017/ml_models

# Logging
LOG_LEVEL=info
LOG_FILE=./logs/ml_service.log

# Performance
MAX_WORKERS=4
CACHE_SIZE=1000
```

### Frontend Environment Variables

```bash
# API Endpoints
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000

# Features
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_NOTIFICATIONS=true

# Analytics (optional)
REACT_APP_GA_TRACKING_ID=your_google_analytics_id
```

## Common Patterns

### Middleware for Protected Routes (Backend)

```javascript
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }

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

// Usage in routes
app.get('/api/users', authMiddleware, adminMiddleware, getAllUsers);
app.get('/api/tasks/user/:id', authMiddleware, getUserTasks);
```

### MongoDB Schema Examples

```javascript
const mongoose = require('mongoose');

// User Schema
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  createdAt: { type: Date, default: Date.now },
  lastLogin: Date,
  loginAttempts: { type: Number, default: 0 },
  isActive: { type: Boolean, default: true }
});

// Task Schema
const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['todo', 'in-progress', 'done'], default: 'todo' },
  priority: { type: String, enum: ['low', 'medium', 'high'], default: 'medium' },
  deadline: Date,
  timeSpent: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: Date
});

// Ticket Schema
const ticketSchema = new mongoose.Schema({
  subject: { type: String, required: true },
  description: { type: String, required: true },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  status: { type: String, enum: ['open', 'in-progress', 'resolved', 'closed'], default: 'open' },
  priority: { type: String, enum: ['low', 'medium', 'high', 'critical'] },
  category: String,
  aiClassification: {
    suggestedCategory: String,
    suggestedPriority: String,
    confidence: Number
  },
  createdAt: { type: Date, default: Date.now },
  resolvedAt: Date
});

module.exports = {
  User: mongoose.model('User', userSchema),
  Task: mongoose.model('Task', taskSchema),
  Ticket: mongoose.model('Ticket', ticketSchema)
};
```

### Real-time Updates with Socket.io

```javascript
// Backend setup
const io = require('socket.io')(server, {
  cors: {
    origin: process.env.FRONTEND_URL,
    methods: ['GET', 'POST']
  }
});

io.on('connection', (socket) => {
  console.log('User connected:', socket.id);

  socket.on('join-room', (userId) => {
    socket.join(`user-${userId}`);
  });

  socket.on('disconnect', () => {
    console.log('User disconnected:', socket.id);
  });
});

// Emit events when data changes
const notifyTaskUpdate = (userId, task) => {
  io.to(`user-${userId}`).emit('task-updated', task);
};

const notifyAnomalyDetected = (userId, anomaly) => {
  io.to(`user-${userId}`).emit('anomaly-alert', anomaly);
};

// Frontend integration
import io from 'socket.io-client';
import { useEffect } from 'react';

const useRealtimeUpdates = (userId, onUpdate) => {
  useEffect(() => {
    const socket = io(process.env.REACT_APP_API_URL);

    socket.emit('join-room', userId);

    socket.on('task-updated', (task) => {
      onUpdate('task', task);
    });

    socket.on('anomaly-alert', (anomaly) => {
      onUpdate('anomaly', anomaly);
    });

    return () => socket.disconnect();
  }, [userId, onUpdate]);
};
```

## Troubleshooting

### Issue: JWT Token Expired

**Problem**: Users getting "Invalid token" errors.

**Solution**:
```javascript
// Implement token refresh logic
const refreshToken = async () => {
  try {
    const refreshToken = localStorage.getItem('refreshToken');
    const response = await axios.post(`${API_URL}/api/auth/refresh`, {
      refreshToken
    });
    localStorage.setItem('authToken', response.data.token);
    return response.data.token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};

// Axios interceptor for automatic token refresh
axios.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;
    
    if (error.response.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      const newToken = await refreshToken();
      originalRequest.headers.Authorization = `Bearer ${newToken}`;
      return axios(originalRequest);
    }
    
    return Promise.reject(error);
  }
);
```

### Issue: ML Service Connection Failed

**Problem**: Backend cannot connect to ML service.

**Solution**:
```javascript
// Add retry logic with fallback
const callMLService = async (endpoint, data, retries = 3) => {
  for (let i = 0; i < retries; i++) {
    try {
      const response = await axios.post(
        `${process.env.ML_SERVICE_URL}${endpoint}`,
        data,
        { timeout: 5000 }
      );
      return response.data;
    } catch (error) {
      if (i === retries - 1) {
        console.error('ML service unavailable, using fallback');
        return getFallbackPrediction(endpoint, data);
      }
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
};

// Fallback predictions
const getFallbackPrediction = (endpoint, data) => {
  if (endpoint.includes('risk-prediction')) {
    return { risk_score: 0.5, risk_level: 'medium', factors: ['ML service unavailable'] };
  }
  if (endpoint.includes('burnout-detection')) {
    return { burnout_risk: 'medium', score: 0.5, recommendations: ['Check back later'] };
  }
  return null;
};
```

### Issue: MongoDB Connection Issues

**Problem**: Cannot connect to MongoDB.

**Solution**:
```javascript
const mongoose = require('mongoose');

const connectDB = async () => {
  const options = {
    useNewUrlParser: true,
    useUnifiedTopology: true,
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
  };

  try {
    await mongoose.connect(process.env.MONGODB_URI, options);
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    // Retry connection after 5 seconds
    setTimeout(connectDB, 5000);
  }
};

// Handle connection events
mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected, attempting to reconnect...');
  connectDB();
});

mongoose.connection.on('error', (err) => {
  console.
