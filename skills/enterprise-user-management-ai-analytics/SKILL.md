---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, and organizational insights
triggers:
  - "help me set up the enterprise user management system"
  - "how do I integrate AI analytics into user management"
  - "create a user dashboard with task tracking"
  - "implement AI-based ticket classification"
  - "set up JWT authentication for user management"
  - "build a kanban board for task management"
  - "add burnout detection and risk prediction features"
  - "configure the ML service for anomaly detection"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines traditional user management with AI-powered insights. It provides role-based access control, task tracking via Kanban boards, support ticket management, and ML-driven analytics including risk prediction, anomaly detection, burnout analysis, and project delay forecasting.

The system consists of three main components:
- **Frontend**: React.js application with user/admin dashboards
- **Backend**: Node.js REST API with JWT authentication and MongoDB
- **ML Service**: FastAPI service using scikit-learn and River for real-time predictions

## Installation

### Prerequisites

- Node.js (v14+)
- Python (v3.8+)
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

```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_ums
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

Create `.env` file:

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

Create `.env` file:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Backend API Reference

### Authentication

```javascript
// Register new user
const register = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/register`, {
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

// Login
const login = async (credentials) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
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

// Get authenticated user
const getUser = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/me`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Create user
const createUser = async (userData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(userData)
  });
  return response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// Get user tasks
const getUserTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Create task
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority,
      dueDate: taskData.dueDate,
      status: 'todo'
    })
  });
  return response.json();
};

// Update task status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};

// Track time on task
const trackTime = async (taskId, timeData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      startTime: timeData.startTime,
      endTime: timeData.endTime,
      duration: timeData.duration
    })
  });
  return response.json();
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      category: ticketData.category,
      priority: ticketData.priority
    })
  });
  return response.json();
};

// Get all tickets (admin)
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update ticket status
const updateTicketStatus = async (ticketId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets/${ticketId}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status })
  });
  return response.json();
};
```

## ML Service API Reference

### AI-Powered Ticket Classification

```javascript
// Classify ticket using AI
const classifyTicket = async (ticketText) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/api/ml/classify-ticket`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      description: ticketText
    })
  });
  const data = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.87 }
  return data;
};
```

### Risk Prediction

```javascript
// Predict user risk based on behavior
const predictRisk = async (userId, behaviorData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/api/ml/predict-risk`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      loginFrequency: behaviorData.loginFrequency,
      taskCompletionRate: behaviorData.taskCompletionRate,
      overdueTasksCount: behaviorData.overdueTasksCount,
      supportTicketsCount: behaviorData.supportTicketsCount,
      averageResponseTime: behaviorData.averageResponseTime
    })
  });
  const data = await response.json();
  // Returns: { riskLevel: 'medium', riskScore: 0.65, factors: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// Detect anomalous user behavior
const detectAnomaly = async (userActivity) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/api/ml/detect-anomaly`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userActivity.userId,
      loginTime: userActivity.loginTime,
      ipAddress: userActivity.ipAddress,
      activityCount: userActivity.activityCount,
      dataAccessPatterns: userActivity.dataAccessPatterns
    })
  });
  const data = await response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.92, reasons: [...] }
  return data;
};
```

### Burnout Detection

```javascript
// Analyze user for burnout risk
const detectBurnout = async (userId, workloadData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/api/ml/detect-burnout`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      weeklyWorkHours: workloadData.weeklyWorkHours,
      overtimeHours: workloadData.overtimeHours,
      taskCount: workloadData.taskCount,
      missedDeadlines: workloadData.missedDeadlines,
      stressLevel: workloadData.stressLevel
    })
  });
  const data = await response.json();
  // Returns: { burnoutRisk: 'high', score: 0.78, recommendations: [...] }
  return data;
};
```

### Predictive Project Insights

```javascript
// Predict project delay probability
const predictProjectDelay = async (projectData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/api/ml/predict-delay`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.projectId,
      tasksCompleted: projectData.tasksCompleted,
      totalTasks: projectData.totalTasks,
      daysRemaining: projectData.daysRemaining,
      teamSize: projectData.teamSize,
      complexityScore: projectData.complexityScore
    })
  });
  const data = await response.json();
  // Returns: { delayProbability: 0.73, estimatedDelay: 5, recommendations: [...] }
  return data;
};
```

## Common Patterns

### Protected Route Component

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  const user = JSON.parse(localStorage.getItem('user') || '{}');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (adminOnly && user.role !== 'admin') {
    return <Navigate to="/dashboard" />;
  }

  return children;
};

export default ProtectedRoute;
```

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    loadTasks();
  }, []);

  const loadTasks = async () => {
    const allTasks = await getUserTasks();
    const grouped = {
      todo: allTasks.filter(t => t.status === 'todo'),
      inProgress: allTasks.filter(t => t.status === 'in_progress'),
      done: allTasks.filter(t => t.status === 'done')
    };
    setTasks(grouped);
  };

  const moveTask = async (taskId, newStatus) => {
    await updateTaskStatus(taskId, newStatus);
    loadTasks();
  };

  const onDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const onDrop = async (e, status) => {
    const taskId = e.dataTransfer.getData('taskId');
    await moveTask(taskId, status);
  };

  return (
    <div className="kanban-board">
      {['todo', 'inProgress', 'done'].map(status => (
        <div
          key={status}
          className="kanban-column"
          onDragOver={(e) => e.preventDefault()}
          onDrop={(e) => onDrop(e, status)}
        >
          <h3>{status.replace(/([A-Z])/g, ' $1').toUpperCase()}</h3>
          {tasks[status].map(task => (
            <div
              key={task._id}
              className="task-card"
              draggable
              onDragStart={(e) => onDragStart(e, task._id)}
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

### Admin Dashboard with AI Insights

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [users, setUsers] = useState([]);
  const [analytics, setAnalytics] = useState({});
  const [alerts, setAlerts] = useState([]);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    const usersData = await getAllUsers();
    setUsers(usersData);

    // Get AI insights for each user
    const insights = await Promise.all(
      usersData.map(async (user) => {
        const risk = await predictRisk(user._id, {
          loginFrequency: user.loginFrequency,
          taskCompletionRate: user.taskCompletionRate,
          overdueTasksCount: user.overdueTasksCount,
          supportTicketsCount: user.supportTicketsCount,
          averageResponseTime: user.averageResponseTime
        });

        const burnout = await detectBurnout(user._id, {
          weeklyWorkHours: user.weeklyWorkHours,
          overtimeHours: user.overtimeHours,
          taskCount: user.taskCount,
          missedDeadlines: user.missedDeadlines,
          stressLevel: user.stressLevel
        });

        return { userId: user._id, risk, burnout };
      })
    );

    setAnalytics(insights);

    // Filter high-risk alerts
    const highRiskAlerts = insights
      .filter(i => i.risk.riskLevel === 'high' || i.burnout.burnoutRisk === 'high')
      .map(i => ({
        userId: i.userId,
        type: i.risk.riskLevel === 'high' ? 'risk' : 'burnout',
        message: `User ${i.userId} requires attention`
      }));

    setAlerts(highRiskAlerts);
  };

  return (
    <div className="admin-dashboard">
      <h1>Admin Dashboard</h1>

      <section className="alerts">
        <h2>AI Alerts</h2>
        {alerts.map((alert, idx) => (
          <div key={idx} className={`alert ${alert.type}`}>
            {alert.message}
          </div>
        ))}
      </section>

      <section className="user-analytics">
        <h2>User Analytics</h2>
        <table>
          <thead>
            <tr>
              <th>User</th>
              <th>Risk Level</th>
              <th>Burnout Risk</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {users.map(user => {
              const userAnalytics = analytics.find(a => a.userId === user._id);
              return (
                <tr key={user._id}>
                  <td>{user.name}</td>
                  <td className={userAnalytics?.risk.riskLevel}>
                    {userAnalytics?.risk.riskLevel || 'N/A'}
                  </td>
                  <td className={userAnalytics?.burnout.burnoutRisk}>
                    {userAnalytics?.burnout.burnoutRisk || 'N/A'}
                  </td>
                  <td>
                    <button onClick={() => viewUserDetails(user._id)}>View</button>
                  </td>
                </tr>
              );
            })}
          </tbody>
        </table>
      </section>
    </div>
  );
};

export default AdminDashboard;
```

### Smart Ticket Creation with AI

```javascript
import React, { useState } from 'react';

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    title: '',
    description: '',
    category: '',
    priority: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);

  const handleDescriptionChange = async (e) => {
    const description = e.target.value;
    setFormData({ ...formData, description });

    // Get AI suggestions after user stops typing
    if (description.length > 20) {
      const suggestions = await classifyTicket(description);
      setAiSuggestions(suggestions);
      
      // Auto-fill if user hasn't selected
      if (!formData.category && !formData.priority) {
        setFormData(prev => ({
          ...prev,
          category: suggestions.category,
          priority: suggestions.priority
        }));
      }
    }
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    await createTicket(formData);
    // Reset form and navigate
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Title"
        value={formData.title}
        onChange={(e) => setFormData({ ...formData, title: e.target.value })}
        required
      />

      <textarea
        placeholder="Description"
        value={formData.description}
        onChange={handleDescriptionChange}
        required
      />

      {aiSuggestions && (
        <div className="ai-suggestions">
          <p>AI Suggestions:</p>
          <p>Category: {aiSuggestions.category} (Confidence: {(aiSuggestions.confidence * 100).toFixed(0)}%)</p>
          <p>Priority: {aiSuggestions.priority}</p>
        </div>
      )}

      <select
        value={formData.category}
        onChange={(e) => setFormData({ ...formData, category: e.target.value })}
        required
      >
        <option value="">Select Category</option>
        <option value="technical">Technical</option>
        <option value="billing">Billing</option>
        <option value="general">General</option>
        <option value="urgent">Urgent</option>
      </select>

      <select
        value={formData.priority}
        onChange={(e) => setFormData({ ...formData, priority: e.target.value })}
        required
      >
        <option value="">Select Priority</option>
        <option value="low">Low</option>
        <option value="medium">Medium</option>
        <option value="high">High</option>
        <option value="critical">Critical</option>
      </select>

      <button type="submit">Create Ticket</button>
    </form>
  );
};

export default CreateTicket;
```

## Configuration

### Backend Environment Variables

```env
# Server
PORT=5000
NODE_ENV=production

# Database
MONGODB_URI=mongodb+srv://username:password@cluster.mongodb.net/database

# JWT
JWT_SECRET=your_strong_secret_key_here
JWT_EXPIRE=7d

# Services
ML_SERVICE_URL=http://ml-service:8000

# CORS
CORS_ORIGIN=https://your-frontend-domain.com

# Logging
LOG_LEVEL=info
```

### ML Service Configuration

```env
# Model paths
MODEL_PATH=/app/models
TICKET_CLASSIFIER_PATH=/app/models/ticket_classifier.pkl
RISK_PREDICTOR_PATH=/app/models/risk_predictor.pkl

# API
ML_API_PORT=8000
LOG_LEVEL=INFO

# Performance
MAX_WORKERS=4
BATCH_SIZE=32
```

### Frontend Environment Variables

```env
REACT_APP_API_URL=https://api.your-domain.com
REACT_APP_ML_URL=https://ml.your-domain.com
REACT_APP_ENV=production
```

## Troubleshooting

### Authentication Issues

**Problem**: JWT token expired or invalid

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
    localStorage.removeItem('token');
    window.location.href = '/login';
  }
};

// Axios interceptor for automatic token refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    if (error.response.status === 401) {
      const newToken = await refreshToken();
      error.config.headers['Authorization'] = `Bearer ${newToken}`;
      return axios.request(error.config);
    }
    return Promise.reject(error);
  }
);
```

### ML Service Connection Issues

**Problem**: ML service unreachable or timeout

```javascript
// Add retry logic for ML requests
const callMLServiceWithRetry = async (endpoint, data, maxRetries = 3) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(`${process.env.REACT_APP_ML_URL}${endpoint}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
        timeout: 5000
      });
      return await response.json();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
};
```

### Database Connection Issues

**Problem**: MongoDB connection failed

```javascript
// backend/config/db.js
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
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Performance Optimization

**Problem**: Slow dashboard loading with many users

```javascript
// Implement pagination for user list
const getUsersPaginated = async (page = 1, limit = 20) => {
  const token = localStorage.getItem('token');
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/api/users?page=${page}&limit=${limit}`,
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.json();
};

// Cache AI predictions
const cachedPredictions = new Map();

const getCachedPrediction = async (userId, predictionType, dataFn) => {
  const key = `${userId}-${predictionType}`;
  const cached = cachedPredictions.get(key);
  
  if (cached && Date.now() - cached.timestamp < 300000) { // 5 min cache
    return cached.data;
  }
  
  const data = await dataFn();
  cachedPredictions.set(key, { data, timestamp: Date.now() });
  return data;
};
```
