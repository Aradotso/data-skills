---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket management, risk detection, and burnout analysis
triggers:
  - "set up enterprise user management system with AI analytics"
  - "implement AI-powered user management and task tracking"
  - "create admin dashboard with user role management"
  - "build kanban board with time tracking features"
  - "integrate AI ticket classification and risk prediction"
  - "add burnout detection and anomaly detection to user system"
  - "develop support ticket system with AI routing"
  - "configure JWT authentication for user management app"
---

# Enterprise User Management AI Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack application that combines user management, task tracking, and support ticket handling with AI-powered insights. The system provides risk detection, anomaly detection, burnout analysis, and predictive project insights to help organizations improve productivity and automate workflows.

**Key Components:**
- **Frontend**: React.js application with admin and user dashboards
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI service with scikit-learn and River for online learning
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

Ensure you have the following installed:
- Node.js (v14+)
- Python (v3.8+)
- MongoDB (running locally or remote connection)

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

```bash
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=24h
```

Start backend server:

```bash
npm start
# Backend runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
```

Start ML service:

```bash
uvicorn main:app --reload
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install
```

Create `.env` file in frontend directory:

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
# Frontend runs at http://localhost:3000
```

## Architecture

### Backend API Structure

The Node.js backend provides RESTful endpoints for:
- User authentication and management
- Task CRUD operations
- Support ticket handling
- Admin dashboard analytics

### Frontend Components

Key React components include:
- `AdminDashboard`: User management, analytics, audit logs
- `UserDashboard`: Personal tasks, notifications, performance
- `KanbanBoard`: Drag-and-drop task management
- `TicketSystem`: Support request tracking
- `AIInsights`: AI-powered analytics display

### ML Service Endpoints

FastAPI service provides AI capabilities:
- Ticket classification and routing
- Risk prediction
- Anomaly detection
- Burnout analysis
- Project delay prediction

## Backend API Examples

### User Authentication

```javascript
// Register new user
const registerUser = async (userData) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/register`, {
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
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Registration error:', error);
    throw error;
  }
};

// Login user
const loginUser = async (credentials) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/login`, {
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
  } catch (error) {
    console.error('Login error:', error);
    throw error;
  }
};
```

### Task Management

```javascript
// Create authenticated request helper
const authFetch = async (url, options = {}) => {
  const token = localStorage.getItem('token');
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
    ...options.headers
  };
  return fetch(url, { ...options, headers });
};

// Get all tasks for user
const getUserTasks = async () => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/tasks`);
  return response.json();
};

// Create new task
const createTask = async (taskData) => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
    method: 'POST',
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      status: taskData.status || 'To Do',
      priority: taskData.priority || 'Medium',
      assignedTo: taskData.assignedTo,
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Update task status
const updateTaskStatus = async (taskId, newStatus) => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
    method: 'PUT',
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};

// Delete task
const deleteTask = async (taskId) => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`, {
    method: 'DELETE'
  });
  return response.json();
};
```

### Support Ticket System

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
    method: 'POST',
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// Get all tickets (admin)
const getAllTickets = async () => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/tickets/admin`);
  return response.json();
};

// Update ticket status
const updateTicketStatus = async (ticketId, status) => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/tickets/${ticketId}`, {
    method: 'PUT',
    body: JSON.stringify({ status })
  });
  return response.json();
};
```

### Admin User Management

```javascript
// Get all users (admin only)
const getAllUsers = async () => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/admin/users`);
  return response.json();
};

// Update user role
const updateUserRole = async (userId, newRole) => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/admin/users/${userId}`, {
    method: 'PUT',
    body: JSON.stringify({ role: newRole })
  });
  return response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/admin/users/${userId}`, {
    method: 'DELETE'
  });
  return response.json();
};

// Get audit logs
const getAuditLogs = async () => {
  const response = await authFetch(`${process.env.REACT_APP_API_URL}/api/admin/audit-logs`);
  return response.json();
};
```

## ML Service Integration

### AI Ticket Classification

```javascript
// Classify and route ticket using AI
const classifyTicket = async (ticketText) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        text: ticketText,
        subject: ticketText.split('\n')[0]
      })
    });
    const data = await response.json();
    // Returns: { category: 'Technical', priority: 'High', suggested_assignee: 'IT Team' }
    return data;
  } catch (error) {
    console.error('Ticket classification error:', error);
    throw error;
  }
};
```

### Risk Prediction

```javascript
// Predict user risk score based on behavior
const predictUserRisk = async (userId, behaviorData) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/risk-prediction`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        user_id: userId,
        login_frequency: behaviorData.loginFrequency,
        failed_login_attempts: behaviorData.failedLogins,
        unusual_activity_hours: behaviorData.unusualHours,
        data_access_volume: behaviorData.dataAccess
      })
    });
    const data = await response.json();
    // Returns: { risk_score: 0.75, risk_level: 'High', factors: [...] }
    return data;
  } catch (error) {
    console.error('Risk prediction error:', error);
    throw error;
  }
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomalies = async (userActivity) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/anomaly-detection`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        user_id: userActivity.userId,
        timestamp: userActivity.timestamp,
        action: userActivity.action,
        resource: userActivity.resource,
        ip_address: userActivity.ipAddress
      })
    });
    const data = await response.json();
    // Returns: { is_anomaly: true, confidence: 0.92, anomaly_type: 'Unusual Access Pattern' }
    return data;
  } catch (error) {
    console.error('Anomaly detection error:', error);
    throw error;
  }
};
```

### Burnout Detection

```javascript
// Analyze user workload for burnout risk
const detectBurnout = async (userId, workloadData) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-detection`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        user_id: userId,
        tasks_completed: workloadData.tasksCompleted,
        average_working_hours: workloadData.avgHours,
        overtime_hours: workloadData.overtime,
        task_completion_rate: workloadData.completionRate,
        stress_indicators: workloadData.stressLevel
      })
    });
    const data = await response.json();
    // Returns: { burnout_risk: 'High', score: 0.82, recommendations: [...] }
    return data;
  } catch (error) {
    console.error('Burnout detection error:', error);
    throw error;
  }
};
```

### Predictive Project Insights

```javascript
// Predict project delay probability
const predictProjectDelay = async (projectData) => {
  try {
    const response = await fetch(`${process.env.REACT_APP_ML_API_URL}/api/ml/project-prediction`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        project_id: projectData.projectId,
        total_tasks: projectData.totalTasks,
        completed_tasks: projectData.completedTasks,
        days_remaining: projectData.daysRemaining,
        team_size: projectData.teamSize,
        average_velocity: projectData.avgVelocity
      })
    });
    const data = await response.json();
    // Returns: { delay_probability: 0.65, estimated_delay_days: 5, factors: [...] }
    return data;
  } catch (error) {
    console.error('Project prediction error:', error);
    throw error;
  }
};
```

## React Component Examples

### Kanban Board Component

```javascript
import React, { useState, useEffect } from 'react';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ 'To Do': [], 'In Progress': [], 'Done': [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const data = await getUserTasks();
    const organized = { 'To Do': [], 'In Progress': [], 'Done': [] };
    data.forEach(task => {
      organized[task.status].push(task);
    });
    setTasks(organized);
  };

  const handleDragStart = (e, taskId) => {
    e.dataTransfer.setData('taskId', taskId);
  };

  const handleDrop = async (e, newStatus) => {
    e.preventDefault();
    const taskId = e.dataTransfer.getData('taskId');
    await updateTaskStatus(taskId, newStatus);
    fetchTasks(); // Refresh
  };

  const handleDragOver = (e) => {
    e.preventDefault();
  };

  return (
    <div className="kanban-board">
      {Object.keys(tasks).map(status => (
        <div 
          key={status} 
          className="kanban-column"
          onDrop={(e) => handleDrop(e, status)}
          onDragOver={handleDragOver}
        >
          <h3>{status}</h3>
          {tasks[status].map(task => (
            <div 
              key={task._id} 
              className="task-card"
              draggable
              onDragStart={(e) => handleDragStart(e, task._id)}
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

### AI Insights Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const AIInsightsDashboard = ({ userId }) => {
  const [insights, setInsights] = useState({
    riskScore: null,
    burnoutRisk: null,
    anomalies: []
  });

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    try {
      // Fetch user behavior data first
      const behaviorResponse = await authFetch(
        `${process.env.REACT_APP_API_URL}/api/users/${userId}/behavior`
      );
      const behaviorData = await behaviorResponse.json();

      // Get AI predictions
      const riskData = await predictUserRisk(userId, behaviorData);
      const burnoutData = await detectBurnout(userId, behaviorData.workload);

      setInsights({
        riskScore: riskData,
        burnoutRisk: burnoutData,
        anomalies: behaviorData.recentAnomalies || []
      });
    } catch (error) {
      console.error('Error fetching AI insights:', error);
    }
  };

  return (
    <div className="ai-insights">
      <h2>AI-Powered Insights</h2>
      
      {insights.riskScore && (
        <div className={`insight-card risk-${insights.riskScore.risk_level}`}>
          <h3>Security Risk Score</h3>
          <div className="score">{(insights.riskScore.risk_score * 100).toFixed(0)}%</div>
          <p>Risk Level: {insights.riskScore.risk_level}</p>
          <ul>
            {insights.riskScore.factors.map((factor, idx) => (
              <li key={idx}>{factor}</li>
            ))}
          </ul>
        </div>
      )}

      {insights.burnoutRisk && (
        <div className={`insight-card burnout-${insights.burnoutRisk.burnout_risk}`}>
          <h3>Burnout Risk Assessment</h3>
          <p>Risk: {insights.burnoutRisk.burnout_risk}</p>
          <div className="recommendations">
            <h4>Recommendations:</h4>
            <ul>
              {insights.burnoutRisk.recommendations.map((rec, idx) => (
                <li key={idx}>{rec}</li>
              ))}
            </ul>
          </div>
        </div>
      )}

      {insights.anomalies.length > 0 && (
        <div className="insight-card anomalies">
          <h3>Recent Anomalies Detected</h3>
          <ul>
            {insights.anomalies.map((anomaly, idx) => (
              <li key={idx}>
                <strong>{anomaly.type}:</strong> {anomaly.description}
                <span className="timestamp">{new Date(anomaly.timestamp).toLocaleString()}</span>
              </li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
};

export default AIInsightsDashboard;
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const [intervalId, setIntervalId] = useState(null);

  useEffect(() => {
    return () => {
      if (intervalId) clearInterval(intervalId);
    };
  }, [intervalId]);

  const startTimer = () => {
    setIsRunning(true);
    const id = setInterval(() => {
      setElapsedTime(prev => prev + 1);
    }, 1000);
    setIntervalId(id);
  };

  const stopTimer = async () => {
    setIsRunning(false);
    if (intervalId) clearInterval(intervalId);
    
    // Save time entry
    await authFetch(`${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`, {
      method: 'POST',
      body: JSON.stringify({ duration: elapsedTime })
    });
  };

  const resetTimer = () => {
    setElapsedTime(0);
    setIsRunning(false);
    if (intervalId) clearInterval(intervalId);
  };

  const formatTime = (seconds) => {
    const hrs = Math.floor(seconds / 3600);
    const mins = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hrs.toString().padStart(2, '0')}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="time-tracker">
      <div className="timer-display">{formatTime(elapsedTime)}</div>
      <div className="timer-controls">
        {!isRunning ? (
          <button onClick={startTimer}>Start</button>
        ) : (
          <button onClick={stopTimer}>Stop</button>
        )}
        <button onClick={resetTimer}>Reset</button>
      </div>
    </div>
  );
};

export default TimeTracker;
```

## Configuration

### Environment Variables

**Backend (.env)**:
```bash
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_secure_secret_key
JWT_EXPIRE=24h
BCRYPT_ROUNDS=10
NODE_ENV=development
```

**Frontend (.env)**:
```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_AI=true
```

**ML Service (.env)**:
```bash
MODEL_PATH=./models
LOG_LEVEL=INFO
ANOMALY_THRESHOLD=0.85
RISK_THRESHOLD=0.70
BURNOUT_THRESHOLD=0.75
```

### MongoDB Schema Examples

**User Schema**:
```javascript
const userSchema = {
  name: String,
  email: { type: String, unique: true },
  password: String, // hashed
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  createdAt: Date,
  lastLogin: Date,
  isActive: Boolean
};
```

**Task Schema**:
```javascript
const taskSchema = {
  title: String,
  description: String,
  status: { type: String, enum: ['To Do', 'In Progress', 'Done'] },
  priority: { type: String, enum: ['Low', 'Medium', 'High'] },
  assignedTo: { type: ObjectId, ref: 'User' },
  createdBy: { type: ObjectId, ref: 'User' },
  dueDate: Date,
  timeTracked: Number, // seconds
  createdAt: Date,
  updatedAt: Date
};
```

**Ticket Schema**:
```javascript
const ticketSchema = {
  subject: String,
  description: String,
  status: { type: String, enum: ['Open', 'In Progress', 'Resolved', 'Closed'] },
  priority: { type: String, enum: ['Low', 'Medium', 'High', 'Critical'] },
  category: String, // AI-classified
  assignedTo: { type: ObjectId, ref: 'User' },
  createdBy: { type: ObjectId, ref: 'User' },
  createdAt: Date,
  resolvedAt: Date
};
```

## Common Patterns

### Protected Routes

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const token = localStorage.getItem('token');
  const userRole = localStorage.getItem('userRole');

  if (!token) {
    return <Navigate to="/login" />;
  }

  if (requiredRole && userRole !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }

  return children;
};

// Usage
<Route path="/admin" element={
  <ProtectedRoute requiredRole="admin">
    <AdminDashboard />
  </ProtectedRoute>
} />
```

### API Error Handling

```javascript
const handleAPIError = (error) => {
  if (error.response) {
    // Server responded with error
    switch (error.response.status) {
      case 401:
        // Unauthorized - redirect to login
        localStorage.removeItem('token');
        window.location.href = '/login';
        break;
      case 403:
        // Forbidden
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
    // No response received
    alert('Network error. Please check your connection');
  } else {
    alert('An unexpected error occurred');
  }
};
```

### Real-time Notifications

```javascript
import { useEffect, useState } from 'react';

const useNotifications = () => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const pollNotifications = async () => {
      try {
        const response = await authFetch(
          `${process.env.REACT_APP_API_URL}/api/notifications`
        );
        const data = await response.json();
        setNotifications(data);
      } catch (error) {
        console.error('Error fetching notifications:', error);
      }
    };

    // Poll every 30 seconds
    const interval = setInterval(pollNotifications, 30000);
    pollNotifications(); // Initial fetch

    return () => clearInterval(interval);
  }, []);

  const markAsRead = async (notificationId) => {
    await authFetch(
      `${process.env.REACT_APP_API_URL}/api/notifications/${notificationId}/read`,
      { method: 'PUT' }
    );
    setNotifications(prev => 
      prev.map(n => n._id === notificationId ? { ...n, read: true } : n)
    );
  };

  return { notifications, markAsRead };
};

export default useNotifications;
```

## Troubleshooting

### Common Issues

**JWT Token Expiration**:
```javascript
// Implement token refresh logic
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
    localStorage.removeItem('token');
    window.location.href = '/login';
  }
};
```

**MongoDB Connection Issues**:
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
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

**CORS Configuration**:
```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

**ML Service Model Loading**:
```python
# ml-service/main.py
import os
from fastapi import FastAPI
import pickle

app = FastAPI()

# Load models on startup
@app.on_event("startup")
async def load_models():
    global risk_model, burnout_model, ticket_classifier
    
    model_path = os.getenv('MODEL_PATH', './models')
    
    try:
        with open(f'{model_path}/risk_model.pkl', 'rb') as f:
            risk_model = pickle.load(f)
        with open(f'{model_path}/burnout_model.pkl', 'rb') as f:
            burnout_model = pickle.load(f)
        with open(f'{model_path}/ticket_classifier.pkl', 'rb') as f:
            ticket_classifier = pickle.load(f)
        print("Models loaded successfully")
    except FileNotFoundError:
        print("Warning: Model files not found. Using default models.")
        # Initialize default models here
```

**Task Status Update Conflicts**:
```javascript
// Implement optimistic locking
const updateTaskWithRetry = async (taskId, updates, version) => {
  try {
    const response = await authFetch(
      `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}`,
      {
        method: 'PUT',
        body: JSON.stringify({ ...updates, version })
      }
    );
    
    if (response.status === 409) {
      // Conflict - version mismatch
      alert('This task was updated by another user. Please refresh.');
      return null;
    }
    
    return response.json();
  } catch (error) {
    console.error('Update failed:', error);
    throw error;
  }
};
```

## Testing

### API Testing Example

```javascript
// tests/api.test.js
const request = require('supertest');
const app = require('../server');

describe('User API', () => {
  let token;

  beforeAll(async () => {
    // Login to get token
    const response = await request(app)
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'password123' });
    token = response.body.token;
  });

  test('GET /api/tasks should return user tasks', async () => {
    const response = await request(app)
      .get('/api/tasks')
      .set('Authorization', `Bearer ${token}`);
    
    expect(response.status).toBe(200);
    expect(Array.isArray(response.body)).toBe(true);
  });

  test('POST /api/tasks should create new task', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .set('Authorization', `Bearer ${token}`)
      .send({
        title: 'Test Task',
        description: 'Test Description',
        status: 'To Do'
      });
    
    expect(response.status).toBe(201);
    expect(response.body.title).toBe('Test Task');
  });
});
```

This skill provides comprehensive guidance for using the Enterprise User Management System with AI Analytics, covering installation, API integration, component development, and troubleshooting for effective AI-powered user and task management.
