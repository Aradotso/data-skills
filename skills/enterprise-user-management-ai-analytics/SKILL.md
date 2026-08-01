---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement role-based access control with AI insights"
  - "configure user task management with ML predictions"
  - "add AI ticket classification and routing"
  - "build admin dashboard with anomaly detection"
  - "create user management system with burnout analysis"
  - "deploy enterprise management with predictive analytics"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that provides centralized user, task, and support ticket management enhanced with machine learning capabilities. The system features AI-driven risk detection, anomaly detection, burnout analysis, and predictive project insights to improve organizational productivity and decision-making.

**Key Components:**
- **Frontend**: React.js dashboard with Kanban board and time tracking
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI-based AI models using scikit-learn and River
- **Database**: MongoDB for data persistence

## Installation

### Prerequisites

```bash
# Node.js 14+ and npm
node --version
npm --version

# Python 3.8+ and pip
python --version
pip --version

# MongoDB
mongod --version
```

### Clone and Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics
```

### Backend Setup

```bash
cd backend
npm install

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
EOF

# Start backend
npm start
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MODEL_PATH=./models
LOG_LEVEL=info
EOF

# Start ML service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

## Project Structure

```
Enterprise-User-Management-System-with-AI-Analytics/
├── frontend/               # React.js application
│   ├── src/
│   │   ├── components/    # UI components
│   │   ├── pages/         # Page components
│   │   ├── services/      # API services
│   │   ├── context/       # React context
│   │   └── utils/         # Utility functions
│   └── package.json
├── backend/               # Node.js API
│   ├── controllers/       # Route controllers
│   ├── models/           # MongoDB models
│   ├── routes/           # API routes
│   ├── middleware/       # Auth & validation
│   └── server.js
└── ml-service/           # FastAPI ML service
    ├── models/           # Trained models
    ├── services/         # ML logic
    ├── main.py          # FastAPI app
    └── requirements.txt
```

## Core API Endpoints

### Authentication

```javascript
// Login
const login = async (email, password) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password })
  });
  const data = await response.json();
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};

// Register
const register = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(userData)
  });
  return await response.json();
};
```

### User Management

```javascript
// Get all users (Admin only)
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(updates)
  });
  return await response.json();
};

// Delete user
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
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
  return await response.json();
};

// Update task status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ status: newStatus })
  });
  return await response.json();
};

// Get user tasks
const getUserTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

### Support Tickets

```javascript
// Create support ticket
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tickets`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return await response.json();
};

// Get tickets with AI classification
const getTicketsWithAI = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch(`${process.env.REACT_APP_API_URL}/tickets?ai=true`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

## AI/ML Integration

### Risk Prediction

```javascript
// Get user risk score
const getUserRiskScore = async (userId) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict/risk`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      features: {
        login_frequency: 15,
        failed_logins: 2,
        avg_session_duration: 120,
        task_completion_rate: 0.85,
        ticket_count: 3
      }
    })
  });
  const data = await response.json();
  // Returns: { risk_score: 0.23, risk_level: "low", factors: [...] }
  return data;
};
```

### Burnout Detection

```javascript
// Analyze user burnout
const analyzeBurnout = async (userId) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict/burnout`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      workload_metrics: {
        hours_worked_week: 45,
        tasks_assigned: 12,
        tasks_completed: 8,
        overtime_hours: 5,
        weekend_work: true,
        avg_response_time: 2.5
      }
    })
  });
  const data = await response.json();
  // Returns: { burnout_probability: 0.67, recommendations: [...] }
  return data;
};
```

### Anomaly Detection

```javascript
// Detect anomalous behavior
const detectAnomalies = async (userId, activityData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/detect/anomaly`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      activities: activityData
    })
  });
  const data = await response.json();
  // Returns: { is_anomalous: true, anomaly_score: 0.89, alerts: [...] }
  return data;
};
```

### Ticket Classification

```javascript
// Auto-classify support ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/classify/ticket`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      metadata: {
        urgency_keywords: true
      }
    })
  });
  const data = await response.json();
  // Returns: { category: "technical", priority: "high", suggested_assignee: "user123" }
  return data;
};
```

### Predictive Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectId, projectData) => {
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict/delay`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectId,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      avg_completion_rate: projectData.avgCompletionRate,
      deadline: projectData.deadline,
      team_size: projectData.teamSize
    })
  });
  const data = await response.json();
  // Returns: { delay_probability: 0.45, estimated_delay_days: 3, recommendations: [...] }
  return data;
};
```

## Backend Implementation Patterns

### User Model (MongoDB)

```javascript
// models/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' },
  department: String,
  status: { type: String, enum: ['active', 'inactive'], default: 'active' },
  lastLogin: Date,
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(candidatePassword) {
  return await bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Task Model

```javascript
// models/Task.js
const mongoose = require('mongoose');

const taskSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  status: { 
    type: String, 
    enum: ['todo', 'in_progress', 'done'], 
    default: 'todo' 
  },
  priority: { 
    type: String, 
    enum: ['low', 'medium', 'high', 'critical'], 
    default: 'medium' 
  },
  assignedTo: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  dueDate: Date,
  timeTracked: { type: Number, default: 0 }, // in minutes
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Task', taskSchema);
```

### Authentication Middleware

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const protect = async (req, res, next) => {
  let token;
  
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.id).select('-password');
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Not authorized to access this route' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    next();
  } else {
    res.status(403).json({ message: 'Admin access required' });
  }
};

module.exports = { protect, adminOnly };
```

### Task Controller

```javascript
// controllers/taskController.js
const Task = require('../models/Task');

exports.createTask = async (req, res) => {
  try {
    const task = await Task.create({
      ...req.body,
      createdBy: req.user.id
    });
    
    res.status(201).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};

exports.getUserTasks = async (req, res) => {
  try {
    const tasks = await Task.find({ assignedTo: req.user.id })
      .populate('createdBy', 'name email')
      .sort('-createdAt');
    
    res.status(200).json({
      success: true,
      count: tasks.length,
      data: tasks
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
};

exports.updateTask = async (req, res) => {
  try {
    const task = await Task.findByIdAndUpdate(
      req.params.id,
      { ...req.body, updatedAt: Date.now() },
      { new: true, runValidators: true }
    );
    
    if (!task) {
      return res.status(404).json({
        success: false,
        error: 'Task not found'
      });
    }
    
    res.status(200).json({
      success: true,
      data: task
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      error: error.message
    });
  }
};
```

## React Component Patterns

### Kanban Board Component

```javascript
// components/KanbanBoard.jsx
import React, { useState, useEffect } from 'react';
import { DragDropContext, Droppable, Draggable } from 'react-beautiful-dnd';

const KanbanBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], in_progress: [], done: [] });
  
  useEffect(() => {
    fetchTasks();
  }, []);
  
  const fetchTasks = async () => {
    const response = await fetch(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`, {
      headers: { 'Authorization': `Bearer ${localStorage.getItem('token')}` }
    });
    const data = await response.json();
    
    const grouped = {
      todo: data.data.filter(t => t.status === 'todo'),
      in_progress: data.data.filter(t => t.status === 'in_progress'),
      done: data.data.filter(t => t.status === 'done')
    };
    
    setTasks(grouped);
  };
  
  const onDragEnd = async (result) => {
    if (!result.destination) return;
    
    const { source, destination, draggableId } = result;
    
    if (source.droppableId !== destination.droppableId) {
      // Update task status
      await fetch(`${process.env.REACT_APP_API_URL}/tasks/${draggableId}`, {
        method: 'PATCH',
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('token')}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ status: destination.droppableId })
      });
      
      fetchTasks();
    }
  };
  
  return (
    <DragDropContext onDragEnd={onDragEnd}>
      <div className="kanban-board">
        {['todo', 'in_progress', 'done'].map(columnId => (
          <Droppable key={columnId} droppableId={columnId}>
            {(provided) => (
              <div
                ref={provided.innerRef}
                {...provided.droppableProps}
                className="kanban-column"
              >
                <h3>{columnId.replace('_', ' ').toUpperCase()}</h3>
                {tasks[columnId].map((task, index) => (
                  <Draggable key={task._id} draggableId={task._id} index={index}>
                    {(provided) => (
                      <div
                        ref={provided.innerRef}
                        {...provided.draggableProps}
                        {...provided.dragHandleProps}
                        className="task-card"
                      >
                        <h4>{task.title}</h4>
                        <p>{task.description}</p>
                        <span className={`priority-${task.priority}`}>
                          {task.priority}
                        </span>
                      </div>
                    )}
                  </Draggable>
                ))}
                {provided.placeholder}
              </div>
            )}
          </Droppable>
        ))}
      </div>
    </DragDropContext>
  );
};

export default KanbanBoard;
```

### AI Analytics Dashboard

```javascript
// components/AIAnalyticsDashboard.jsx
import React, { useState, useEffect } from 'react';

const AIAnalyticsDashboard = ({ userId }) => {
  const [analytics, setAnalytics] = useState({
    riskScore: null,
    burnoutProbability: null,
    anomalies: []
  });
  
  useEffect(() => {
    fetchAIAnalytics();
  }, [userId]);
  
  const fetchAIAnalytics = async () => {
    try {
      // Fetch risk score
      const riskResponse = await fetch(`${process.env.REACT_APP_ML_URL}/predict/risk`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId })
      });
      const riskData = await riskResponse.json();
      
      // Fetch burnout analysis
      const burnoutResponse = await fetch(`${process.env.REACT_APP_ML_URL}/predict/burnout`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId })
      });
      const burnoutData = await burnoutResponse.json();
      
      setAnalytics({
        riskScore: riskData,
        burnoutProbability: burnoutData,
        anomalies: []
      });
    } catch (error) {
      console.error('Error fetching AI analytics:', error);
    }
  };
  
  return (
    <div className="ai-analytics-dashboard">
      <div className="analytics-card">
        <h3>Risk Assessment</h3>
        {analytics.riskScore && (
          <div>
            <div className={`risk-level-${analytics.riskScore.risk_level}`}>
              Risk Level: {analytics.riskScore.risk_level}
            </div>
            <div className="risk-score">
              Score: {(analytics.riskScore.risk_score * 100).toFixed(1)}%
            </div>
          </div>
        )}
      </div>
      
      <div className="analytics-card">
        <h3>Burnout Analysis</h3>
        {analytics.burnoutProbability && (
          <div>
            <div className="burnout-meter">
              Probability: {(analytics.burnoutProbability.burnout_probability * 100).toFixed(1)}%
            </div>
            {analytics.burnoutProbability.recommendations && (
              <ul>
                {analytics.burnoutProbability.recommendations.map((rec, idx) => (
                  <li key={idx}>{rec}</li>
                ))}
              </ul>
            )}
          </div>
        )}
      </div>
    </div>
  );
};

export default AIAnalyticsDashboard;
```

## Configuration

### Environment Variables

**Backend (.env)**
```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# JWT
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=${SMTP_HOST}
SMTP_PORT=${SMTP_PORT}
SMTP_USER=${SMTP_USER}
SMTP_PASS=${SMTP_PASS}
```

**Frontend (.env)**
```bash
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
REACT_APP_ENABLE_AI=true
```

**ML Service (.env)**
```bash
MODEL_PATH=./models
LOG_LEVEL=info
ANOMALY_THRESHOLD=0.75
RISK_THRESHOLD=0.6
```

## Common Workflows

### Complete User Task Workflow

```javascript
// services/taskService.js
export const completeTaskWorkflow = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  
  // 1. Update task status
  await fetch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      status: 'done',
      timeTracked: timeSpent
    })
  });
  
  // 2. Log activity for AI analysis
  await fetch(`${process.env.REACT_APP_API_URL}/activity-log`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      action: 'task_completed',
      taskId: taskId,
      timeSpent: timeSpent,
      timestamp: new Date().toISOString()
    })
  });
  
  // 3. Check for burnout indicators
  const burnoutCheck = await fetch(`${process.env.REACT_APP_ML_URL}/predict/burnout`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ 
      user_id: localStorage.getItem('userId'),
      recent_completion: true
    })
  });
  
  const burnoutData = await burnoutCheck.json();
  
  if (burnoutData.burnout_probability > 0.7) {
    // Show warning notification
    return { success: true, warning: 'burnout_risk', data: burnoutData };
  }
  
  return { success: true };
};
```

### Admin Bulk User Management

```javascript
// services/adminService.js
export const bulkUpdateUsers = async (userIds, updates) => {
  const token = localStorage.getItem('token');
  
  const promises = userIds.map(userId =>
    fetch(`${process.env.REACT_APP_API_URL}/users/${userId}`, {
      method: 'PUT',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(updates)
    })
  );
  
  const results = await Promise.allSettled(promises);
  
  const successful = results.filter(r => r.status === 'fulfilled').length;
  const failed = results.filter(r => r.status === 'rejected').length;
  
  return {
    successful,
    failed,
    total: userIds.length
  };
};
```

## Troubleshooting

### JWT Token Expiration

```javascript
// utils/apiClient.js
export const apiClient = async (url, options = {}) => {
  const token = localStorage.getItem('token');
  
  const config = {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`
    }
  };
  
  try {
    const response = await fetch(url, config);
    
    if (response.status === 401) {
      // Token expired, redirect to login
      localStorage.removeItem('token');
      window.location.href = '/login';
      throw new Error('Session expired');
    }
    
    return await response.json();
  } catch (error) {
    console.error('API Error:', error);
    throw error;
  }
};
```

### ML Service Connection Issues

```javascript
// utils/mlServiceHealth.js
export const checkMLServiceHealth = async () => {
  try {
    const response = await fetch(`${process.env.REACT_APP_ML_URL}/health`, {
      timeout: 5000
    });
    
    if (!response.ok) {
      console.error('ML service unhealthy');
      return false;
    }
    
    return true;
  } catch (error) {
    console.error('ML service unavailable:', error);
    return false;
  }
};

// Use with fallback
export const getAIInsights = async (data) => {
  const isHealthy = await checkMLServiceHealth();
  
  if (!isHealthy) {
    // Return mock data or disable AI features
    return { ai_enabled: false, fallback: true };
  }
  
  // Proceed with ML service call
  const response = await fetch(`${process.env.REACT_APP_ML_URL}/predict/risk`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
  });
  
  return await response.json();
};
```

### MongoDB Connection Errors

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    const conn = await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    
    console.log(`MongoDB Connected: ${conn.connection.host}`);
  } catch (error) {
    console.error(`Error: ${error.message}`);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### Performance Optimization

```javascript
// Implement request caching
const cache = new Map();
const CACHE_TTL = 5 * 60 * 1000; // 5 minutes

export const getCachedData = async (key, fetchFunction) => {
  const cached = cache.get(key);
  
  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    return cached.data;
  }
  
  const data = await fetchFunction();
  cache.set(key, { data, timestamp: Date.now() });
  
  return data;
};

// Usage
const analytics = await getCachedData(
  `analytics-${userId}`,
  () => fetchAIAnalytics(userId)
);
```

## Deployment

### Production Build

```bash
# Build frontend
cd frontend
npm run build

# Build backend (if TypeScript)
cd backend
npm run build

# Start production server
NODE_ENV=production npm start
```

### Docker Deployment (Example)

```bash
# Backend
docker build -t enterprise-mgmt-backend ./backend
docker run -p 5000:5000 \
  -e MONGODB_URI=${MONGODB_URI} \
  -e JWT_SECRET=${JWT_SECRET} \
  enterprise-mgmt-backend

# ML Service
docker build -t enterprise-mgmt-ml ./ml-service
docker run -p 8000:8000 enterprise-mgmt-ml

# Frontend
docker build -t enterprise-mgmt-frontend ./frontend
docker run -p 80:80 enterprise-mgmt-
