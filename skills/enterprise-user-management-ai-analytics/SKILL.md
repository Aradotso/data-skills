---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics, task tracking, ticket routing, and risk detection capabilities
triggers:
  - how do I set up the enterprise user management system
  - integrate AI analytics into user management
  - implement ticket classification with machine learning
  - build a user management dashboard with AI insights
  - create task tracking with burnout detection
  - set up JWT authentication for user management
  - deploy AI-powered user analytics system
  - configure anomaly detection for enterprise users
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Enterprise User Management System is a full-stack JavaScript/Python application that combines traditional user management with AI-powered analytics. It provides role-based access control, task tracking with Kanban boards, support ticket management, and intelligent insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

**Key Components:**
- **Frontend**: React.js dashboard for users and admins
- **Backend**: Node.js REST API with JWT authentication
- **ML Service**: FastAPI microservice with scikit-learn and River for online learning
- **Database**: MongoDB for persistent storage

## Installation

### Prerequisites

```bash
# Required: Node.js 14+, Python 3.8+, MongoDB
node --version  # v14.0.0 or higher
python --version  # 3.8 or higher
mongod --version  # MongoDB 4.4+
```

### Clone and Setup

```bash
# Clone the repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Install backend dependencies
cd backend
npm install

# Install ML service dependencies
cd ../ml-service
pip install -r requirements.txt

# Install frontend dependencies
cd ../frontend
npm install
```

### Environment Configuration

**Backend** (`backend/.env`):
```env
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise-user-mgmt
JWT_SECRET=your_jwt_secret_key_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
```

**ML Service** (`ml-service/.env`):
```env
MODEL_PATH=./models
LOG_LEVEL=info
BACKEND_URL=http://localhost:5000
```

**Frontend** (`frontend/.env`):
```env
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_ML_URL=http://localhost:8000
```

## Running the Application

### Start Backend Server

```bash
cd backend
npm start
# Server runs at http://localhost:5000
```

### Start ML Service

```bash
cd ml-service
uvicorn main:app --reload --host 0.0.0.0 --port 8000
# ML service runs at http://localhost:8000
```

### Start Frontend

```bash
cd frontend
npm start
# Frontend runs at http://localhost:3000
```

## Core Backend API Usage

### Authentication

**Register User** (JavaScript):
```javascript
const axios = require('axios');

async function registerUser(userData) {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/register', {
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user', // 'user' or 'admin'
      department: userData.department
    });
    
    return response.data; // { token, user }
  } catch (error) {
    console.error('Registration failed:', error.response.data);
    throw error;
  }
}

// Usage
registerUser({
  name: 'John Doe',
  email: 'john@example.com',
  password: 'securePassword123',
  role: 'user',
  department: 'Engineering'
});
```

**Login**:
```javascript
async function login(email, password) {
  try {
    const response = await axios.post('http://localhost:5000/api/auth/login', {
      email,
      password
    });
    
    // Store token for subsequent requests
    const token = response.data.token;
    localStorage.setItem('authToken', token);
    
    return response.data;
  } catch (error) {
    console.error('Login failed:', error.response.data);
    throw error;
  }
}
```

**Authenticated Request Helper**:
```javascript
function getAuthHeaders() {
  const token = localStorage.getItem('authToken');
  return {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  };
}
```

### User Management (Admin)

**Get All Users**:
```javascript
async function getAllUsers() {
  const response = await axios.get(
    'http://localhost:5000/api/admin/users',
    getAuthHeaders()
  );
  return response.data;
}
```

**Update User**:
```javascript
async function updateUser(userId, updates) {
  const response = await axios.put(
    `http://localhost:5000/api/admin/users/${userId}`,
    {
      name: updates.name,
      email: updates.email,
      role: updates.role,
      department: updates.department,
      status: updates.status // 'active' or 'inactive'
    },
    getAuthHeaders()
  );
  return response.data;
}
```

**Delete User**:
```javascript
async function deleteUser(userId) {
  const response = await axios.delete(
    `http://localhost:5000/api/admin/users/${userId}`,
    getAuthHeaders()
  );
  return response.data;
}
```

### Task Management

**Create Task**:
```javascript
async function createTask(taskData) {
  const response = await axios.post(
    'http://localhost:5000/api/tasks',
    {
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in-progress', 'done'
    },
    getAuthHeaders()
  );
  return response.data;
}
```

**Update Task Status**:
```javascript
async function updateTaskStatus(taskId, status) {
  const response = await axios.patch(
    `http://localhost:5000/api/tasks/${taskId}`,
    { status }, // 'todo', 'in-progress', 'done'
    getAuthHeaders()
  );
  return response.data;
}
```

**Track Time on Task**:
```javascript
async function logTaskTime(taskId, timeSpent) {
  const response = await axios.post(
    `http://localhost:5000/api/tasks/${taskId}/time`,
    {
      duration: timeSpent, // in minutes
      date: new Date().toISOString()
    },
    getAuthHeaders()
  );
  return response.data;
}
```

### Support Tickets

**Create Ticket**:
```javascript
async function createTicket(ticketData) {
  const response = await axios.post(
    'http://localhost:5000/api/tickets',
    {
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority, // 'low', 'medium', 'high', 'critical'
      category: ticketData.category // 'technical', 'hr', 'general'
    },
    getAuthHeaders()
  );
  return response.data;
}
```

**Get User Tickets**:
```javascript
async function getUserTickets(userId) {
  const response = await axios.get(
    `http://localhost:5000/api/tickets/user/${userId}`,
    getAuthHeaders()
  );
  return response.data;
}
```

**Update Ticket Status** (Admin):
```javascript
async function updateTicket(ticketId, updates) {
  const response = await axios.put(
    `http://localhost:5000/api/tickets/${ticketId}`,
    {
      status: updates.status, // 'open', 'in-progress', 'resolved', 'closed'
      assignedTo: updates.assignedTo,
      resolution: updates.resolution
    },
    getAuthHeaders()
  );
  return response.data;
}
```

## ML Service API Integration

### Ticket Classification

**Auto-classify Support Ticket**:
```javascript
async function classifyTicket(ticketText) {
  const response = await axios.post('http://localhost:8000/classify-ticket', {
    text: ticketText,
    subject: 'Ticket subject'
  });
  
  return response.data; 
  // { category: 'technical', priority: 'high', confidence: 0.87 }
}

// Example usage in ticket creation
async function createSmartTicket(ticketData) {
  // Get AI classification
  const classification = await classifyTicket(ticketData.description);
  
  // Create ticket with AI suggestions
  return await axios.post(
    'http://localhost:5000/api/tickets',
    {
      subject: ticketData.subject,
      description: ticketData.description,
      priority: classification.priority,
      category: classification.category,
      aiConfidence: classification.confidence
    },
    getAuthHeaders()
  );
}
```

### Risk Detection

**Analyze User Risk**:
```javascript
async function analyzeUserRisk(userId) {
  const response = await axios.post('http://localhost:8000/analyze-risk', {
    user_id: userId,
    features: {
      task_completion_rate: 0.75,
      average_delay_days: 3,
      ticket_frequency: 8,
      login_anomalies: 2,
      work_hours_variance: 15
    }
  });
  
  return response.data;
  // { risk_score: 0.65, risk_level: 'medium', factors: [...] }
}
```

### Anomaly Detection

**Detect Anomalous Behavior**:
```javascript
async function detectAnomaly(userActivity) {
  const response = await axios.post('http://localhost:8000/detect-anomaly', {
    user_id: userActivity.userId,
    activity_type: userActivity.type, // 'login', 'data_access', 'task_update'
    timestamp: new Date().toISOString(),
    metadata: {
      ip_address: userActivity.ip,
      location: userActivity.location,
      device: userActivity.device
    }
  });
  
  return response.data;
  // { is_anomaly: true, anomaly_score: 0.82, reason: 'Unusual login location' }
}
```

### Burnout Detection

**Check Employee Burnout Risk**:
```javascript
async function checkBurnout(userId) {
  const response = await axios.post('http://localhost:8000/burnout-analysis', {
    user_id: userId,
    metrics: {
      weekly_hours: 55,
      tasks_count: 12,
      overdue_tasks: 4,
      weekend_work_hours: 8,
      consecutive_work_days: 14,
      ticket_response_time: 120 // minutes
    }
  });
  
  return response.data;
  // { burnout_score: 0.73, burnout_risk: 'high', recommendations: [...] }
}
```

### Predictive Insights

**Predict Project Delay**:
```javascript
async function predictProjectDelay(projectData) {
  const response = await axios.post('http://localhost:8000/predict-delay', {
    project_id: projectData.id,
    features: {
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      team_size: projectData.teamSize,
      days_remaining: projectData.daysRemaining,
      average_task_duration: projectData.avgTaskDuration,
      high_priority_tasks: projectData.highPriorityTasks
    }
  });
  
  return response.data;
  // { delay_probability: 0.68, estimated_delay_days: 5, confidence: 0.85 }
}
```

## React Frontend Integration

### Authentication Context

```javascript
// frontend/src/context/AuthContext.js
import React, { createContext, useState, useEffect } from 'react';
import axios from 'axios';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const token = localStorage.getItem('authToken');
    if (token) {
      axios.defaults.headers.common['Authorization'] = `Bearer ${token}`;
      fetchUser();
    } else {
      setLoading(false);
    }
  }, []);

  const fetchUser = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/auth/me`);
      setUser(response.data);
    } catch (error) {
      console.error('Failed to fetch user:', error);
      logout();
    } finally {
      setLoading(false);
    }
  };

  const login = async (email, password) => {
    const response = await axios.post(`${process.env.REACT_APP_API_URL}/auth/login`, {
      email,
      password
    });
    
    localStorage.setItem('authToken', response.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
    setUser(response.data.user);
    return response.data;
  };

  const logout = () => {
    localStorage.removeItem('authToken');
    delete axios.defaults.headers.common['Authorization'];
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, login, logout, loading }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Task Dashboard Component

```javascript
// frontend/src/components/TaskDashboard.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const TaskDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    try {
      const response = await axios.get(`${process.env.REACT_APP_API_URL}/tasks/my-tasks`);
      
      // Group tasks by status
      const grouped = {
        todo: response.data.filter(t => t.status === 'todo'),
        inProgress: response.data.filter(t => t.status === 'in-progress'),
        done: response.data.filter(t => t.status === 'done')
      };
      
      setTasks(grouped);
    } catch (error) {
      console.error('Failed to fetch tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const updateTaskStatus = async (taskId, newStatus) => {
    try {
      await axios.patch(`${process.env.REACT_APP_API_URL}/tasks/${taskId}`, {
        status: newStatus
      });
      fetchTasks(); // Refresh
    } catch (error) {
      console.error('Failed to update task:', error);
    }
  };

  const TaskColumn = ({ title, tasks, status }) => (
    <div className="task-column">
      <h3>{title}</h3>
      {tasks.map(task => (
        <div key={task._id} className="task-card">
          <h4>{task.title}</h4>
          <p>{task.description}</p>
          <span className={`priority-${task.priority}`}>{task.priority}</span>
          <div className="task-actions">
            {status !== 'done' && (
              <button onClick={() => updateTaskStatus(task._id, 
                status === 'todo' ? 'in-progress' : 'done')}>
                Move to {status === 'todo' ? 'In Progress' : 'Done'}
              </button>
            )}
          </div>
        </div>
      ))}
    </div>
  );

  if (loading) return <div>Loading tasks...</div>;

  return (
    <div className="kanban-board">
      <TaskColumn title="To Do" tasks={tasks.todo} status="todo" />
      <TaskColumn title="In Progress" tasks={tasks.inProgress} status="in-progress" />
      <TaskColumn title="Done" tasks={tasks.done} status="done" />
    </div>
  );
};

export default TaskDashboard;
```

### AI Insights Component

```javascript
// frontend/src/components/AIInsights.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const AIInsights = ({ userId }) => {
  const [insights, setInsights] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAIInsights();
  }, [userId]);

  const fetchAIInsights = async () => {
    try {
      const [riskData, burnoutData] = await Promise.all([
        axios.post(`${process.env.REACT_APP_ML_URL}/analyze-risk`, {
          user_id: userId
        }),
        axios.post(`${process.env.REACT_APP_ML_URL}/burnout-analysis`, {
          user_id: userId
        })
      ]);

      setInsights({
        risk: riskData.data,
        burnout: burnoutData.data
      });
    } catch (error) {
      console.error('Failed to fetch AI insights:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Analyzing data...</div>;
  if (!insights) return <div>No insights available</div>;

  return (
    <div className="ai-insights">
      <div className="insight-card">
        <h3>Risk Assessment</h3>
        <div className={`risk-level risk-${insights.risk.risk_level}`}>
          {insights.risk.risk_level.toUpperCase()}
        </div>
        <p>Score: {(insights.risk.risk_score * 100).toFixed(0)}%</p>
      </div>

      <div className="insight-card">
        <h3>Burnout Analysis</h3>
        <div className={`burnout-level burnout-${insights.burnout.burnout_risk}`}>
          {insights.burnout.burnout_risk.toUpperCase()}
        </div>
        <p>Score: {(insights.burnout.burnout_score * 100).toFixed(0)}%</p>
        
        {insights.burnout.recommendations && (
          <ul>
            {insights.burnout.recommendations.map((rec, idx) => (
              <li key={idx}>{rec}</li>
            ))}
          </ul>
        )}
      </div>
    </div>
  );
};

export default AIInsights;
```

## Common Patterns

### Pagination for Large Datasets

```javascript
async function getUsersWithPagination(page = 1, limit = 20) {
  const response = await axios.get(
    `http://localhost:5000/api/admin/users`,
    {
      ...getAuthHeaders(),
      params: { page, limit }
    }
  );
  
  return {
    users: response.data.users,
    total: response.data.total,
    pages: response.data.pages,
    currentPage: response.data.currentPage
  };
}
```

### Real-time Notifications with WebSocket

```javascript
// Frontend WebSocket connection
import io from 'socket.io-client';

const socket = io(process.env.REACT_APP_API_URL, {
  auth: {
    token: localStorage.getItem('authToken')
  }
});

socket.on('task-assigned', (data) => {
  console.log('New task assigned:', data.task);
  // Update UI
});

socket.on('ticket-updated', (data) => {
  console.log('Ticket status changed:', data.ticket);
  // Update UI
});

socket.on('anomaly-detected', (data) => {
  console.log('Security alert:', data.anomaly);
  // Show notification
});
```

### Batch User Operations

```javascript
async function batchUpdateUsers(updates) {
  const response = await axios.post(
    'http://localhost:5000/api/admin/users/batch',
    {
      operations: updates.map(u => ({
        userId: u.id,
        updates: {
          department: u.department,
          role: u.role
        }
      }))
    },
    getAuthHeaders()
  );
  
  return response.data; // { success: 45, failed: 5, errors: [...] }
}
```

### Export Analytics Data

```javascript
async function exportAnalytics(format = 'csv', dateRange) {
  const response = await axios.get(
    'http://localhost:5000/api/admin/analytics/export',
    {
      ...getAuthHeaders(),
      params: {
        format, // 'csv', 'json', 'xlsx'
        startDate: dateRange.start,
        endDate: dateRange.end
      },
      responseType: 'blob'
    }
  );
  
  // Download file
  const url = window.URL.createObjectURL(new Blob([response.data]));
  const link = document.createElement('a');
  link.href = url;
  link.setAttribute('download', `analytics-${Date.now()}.${format}`);
  document.body.appendChild(link);
  link.click();
}
```

## Troubleshooting

### JWT Token Expiration

```javascript
// Axios interceptor for automatic token refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;
    
    if (error.response.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      
      try {
        const refreshToken = localStorage.getItem('refreshToken');
        const response = await axios.post('/api/auth/refresh', { refreshToken });
        
        localStorage.setItem('authToken', response.data.token);
        axios.defaults.headers.common['Authorization'] = `Bearer ${response.data.token}`;
        
        return axios(originalRequest);
      } catch (err) {
        // Redirect to login
        window.location.href = '/login';
        return Promise.reject(err);
      }
    }
    
    return Promise.reject(error);
  }
);
```

### MongoDB Connection Issues

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    
    console.log('MongoDB connected successfully');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

// Handle connection events
mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected. Attempting to reconnect...');
});

module.exports = connectDB;
```

### ML Service Model Loading Errors

```python
# ml-service/main.py
from fastapi import FastAPI, HTTPException
import pickle
import os

app = FastAPI()

# Lazy load models
models = {}

def load_model(model_name):
    if model_name not in models:
        model_path = os.path.join(os.getenv('MODEL_PATH', './models'), f'{model_name}.pkl')
        try:
            with open(model_path, 'rb') as f:
                models[model_name] = pickle.load(f)
        except FileNotFoundError:
            # Initialize with default model if file doesn't exist
            from sklearn.ensemble import RandomForestClassifier
            models[model_name] = RandomForestClassifier()
            print(f"Warning: {model_name} not found, using default model")
    return models[model_name]

@app.post("/classify-ticket")
async def classify_ticket(data: dict):
    try:
        model = load_model('ticket_classifier')
        # Process and predict
        prediction = model.predict([data['text']])
        return {"category": prediction[0]}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### CORS Configuration

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Performance Optimization for Large User Base

```javascript
// Add indexes to MongoDB collections
// backend/models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true, index: true },
  name: { type: String, required: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user', index: true },
  department: { type: String, index: true },
  status: { type: String, enum: ['active', 'inactive'], default: 'active', index: true },
  createdAt: { type: Date, default: Date.now }
});

// Compound index for common queries
userSchema.index({ department: 1, status: 1 });
userSchema.index({ role: 1, createdAt: -1 });

module.exports = mongoose.model('User', userSchema);
```
