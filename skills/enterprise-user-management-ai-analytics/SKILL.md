---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered task analytics, risk detection, and ticket classification using React, Node.js, and FastAPI ML service
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "build user dashboard with task tracking"
  - "implement AI-powered ticket classification"
  - "create admin dashboard with user analytics"
  - "add risk detection and burnout analysis"
  - "configure JWT authentication for user system"
  - "deploy user management with ML service"
---

# Enterprise User Management AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack enterprise user management system with integrated AI analytics for task management, risk detection, anomaly detection, and predictive insights. Built with React frontend, Node.js/Express backend, FastAPI ML service, and MongoDB database.

## What It Does

- **User Management**: Complete CRUD operations with role-based access control (Admin/User)
- **Task Tracking**: Kanban board (To Do → In Progress → Done) with time tracking
- **Support Tickets**: Smart ticket system with AI-powered classification and routing
- **AI Analytics**: Risk prediction, anomaly detection, burnout analysis, project delay prediction
- **Authentication**: JWT-based secure authentication and authorization
- **Dashboards**: Separate admin and user dashboards with real-time analytics

## Installation

### Prerequisites

```bash
# Node.js 14+, Python 3.8+, MongoDB
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

# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
# Server runs at http://localhost:5000
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

uvicorn main:app --reload --port 8000
# ML service runs at http://localhost:8000
```

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

npm start
# Frontend runs at http://localhost:3000
```

## Backend API Reference

### Authentication Endpoints

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      username: userData.username,
      email: userData.email,
      password: userData.password,
      role: userData.role || 'user'
    })
  });
  return response.json();
};

// POST /api/auth/login
const loginUser = async (credentials) => {
  const response = await fetch('http://localhost:5000/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
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

### User Management Endpoints

```javascript
// GET /api/users (Admin only)
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PUT /api/users/:id
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify(updates)
  });
  return response.json();
};

// DELETE /api/users/:id
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
};
```

### Task Management Endpoints

```javascript
// POST /api/tasks
const createTask = async (taskData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.assignedTo,
      priority: taskData.priority, // 'low', 'medium', 'high'
      status: 'todo', // 'todo', 'in_progress', 'done'
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// GET /api/tasks/user/:userId
const getUserTasks = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// PATCH /api/tasks/:id/status
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ status: newStatus })
  });
  return response.json();
};
```

### Support Ticket Endpoints

```javascript
// POST /api/tickets
const createTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      category: ticketData.category,
      priority: ticketData.priority
    })
  });
  return response.json();
};

// GET /api/tickets
const getAllTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

## ML Service API Reference

### AI-Powered Ticket Classification

```javascript
// POST /classify-ticket
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Issue with login",
      description: "Cannot access my account after password reset"
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.85 }
  return result;
};
```

### Risk Prediction

```javascript
// POST /predict-risk
const predictUserRisk = async (userId, behaviorData) => {
  const response = await fetch('http://localhost:8000/predict-risk', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      login_frequency: behaviorData.loginFrequency,
      failed_logins: behaviorData.failedLogins,
      unusual_hours: behaviorData.unusualHours,
      data_access_volume: behaviorData.dataAccessVolume
    })
  });
  const result = await response.json();
  // Returns: { risk_score: 0.72, risk_level: 'medium', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// POST /detect-anomaly
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/detect-anomaly', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: activityData.userId,
      activity_type: activityData.type,
      timestamp: activityData.timestamp,
      ip_address: activityData.ip,
      features: activityData.features
    })
  });
  const result = await response.json();
  // Returns: { is_anomaly: true, anomaly_score: 0.89, reason: 'Unusual login time' }
  return result;
};
```

### Burnout Detection

```javascript
// POST /detect-burnout
const detectBurnout = async (userId, workloadData) => {
  const response = await fetch('http://localhost:8000/detect-burnout', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      user_id: userId,
      tasks_completed: workloadData.tasksCompleted,
      hours_worked: workloadData.hoursWorked,
      overtime_hours: workloadData.overtimeHours,
      task_deadline_misses: workloadData.missedDeadlines,
      stress_indicators: workloadData.stressIndicators
    })
  });
  const result = await response.json();
  // Returns: { burnout_risk: 'high', score: 0.78, recommendations: [...] }
  return result;
};
```

### Project Delay Prediction

```javascript
// POST /predict-delay
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      project_id: projectData.id,
      total_tasks: projectData.totalTasks,
      completed_tasks: projectData.completedTasks,
      team_size: projectData.teamSize,
      days_remaining: projectData.daysRemaining,
      current_velocity: projectData.velocity
    })
  });
  const result = await response.json();
  // Returns: { delay_probability: 0.65, estimated_delay_days: 5, factors: [...] }
  return result;
};
```

## Frontend Components

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [stats, setStats] = useState({});

  useEffect(() => {
    const fetchUserData = async () => {
      const token = localStorage.getItem('token');
      const userId = JSON.parse(atob(token.split('.')[1])).userId;
      
      // Fetch tasks
      const tasksRes = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const tasksData = await tasksRes.json();
      setTasks(tasksData);
      
      // Calculate stats
      setStats({
        total: tasksData.length,
        todo: tasksData.filter(t => t.status === 'todo').length,
        inProgress: tasksData.filter(t => t.status === 'in_progress').length,
        done: tasksData.filter(t => t.status === 'done').length
      });
    };
    
    fetchUserData();
  }, []);

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}/status`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    // Refresh tasks
    window.location.reload();
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      <div className="stats">
        <div className="stat-card">
          <h3>Total Tasks</h3>
          <p>{stats.total}</p>
        </div>
        <div className="stat-card">
          <h3>To Do</h3>
          <p>{stats.todo}</p>
        </div>
        <div className="stat-card">
          <h3>In Progress</h3>
          <p>{stats.inProgress}</p>
        </div>
        <div className="stat-card">
          <h3>Completed</h3>
          <p>{stats.done}</p>
        </div>
      </div>
      
      <div className="kanban-board">
        <div className="column">
          <h2>To Do</h2>
          {tasks.filter(t => t.status === 'todo').map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'in_progress')}>
                Start
              </button>
            </div>
          ))}
        </div>
        
        <div className="column">
          <h2>In Progress</h2>
          {tasks.filter(t => t.status === 'in_progress').map(task => (
            <div key={task._id} className="task-card">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
              <button onClick={() => moveTask(task._id, 'done')}>
                Complete
              </button>
            </div>
          ))}
        </div>
        
        <div className="column">
          <h2>Done</h2>
          {tasks.filter(t => t.status === 'done').map(task => (
            <div key={task._id} className="task-card completed">
              <h4>{task.title}</h4>
              <p>{task.description}</p>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
import React, { useState, useEffect } from 'react';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    users: [],
    riskAlerts: [],
    burnoutAlerts: []
  });

  useEffect(() => {
    const fetchAnalytics = async () => {
      const token = localStorage.getItem('token');
      
      // Fetch all users
      const usersRes = await fetch('http://localhost:5000/api/users', {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const users = await usersRes.json();
      
      // Check risk and burnout for each user
      const riskChecks = await Promise.all(
        users.map(async (user) => {
          const riskRes = await fetch('http://localhost:8000/predict-risk', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              user_id: user._id,
              login_frequency: user.loginFrequency || 0,
              failed_logins: user.failedLogins || 0,
              unusual_hours: user.unusualHours || 0,
              data_access_volume: user.dataAccess || 0
            })
          });
          return { user: user.username, ...(await riskRes.json()) };
        })
      );
      
      setAnalytics({
        users,
        riskAlerts: riskChecks.filter(r => r.risk_level === 'high'),
        burnoutAlerts: [] // Similar pattern for burnout
      });
    };
    
    fetchAnalytics();
  }, []);

  return (
    <div className="admin-analytics">
      <h1>Admin Analytics</h1>
      
      <section className="alerts">
        <h2>Risk Alerts</h2>
        {analytics.riskAlerts.map((alert, idx) => (
          <div key={idx} className="alert-card risk">
            <h3>{alert.user}</h3>
            <p>Risk Score: {alert.risk_score}</p>
            <p>Level: {alert.risk_level}</p>
            <ul>
              {alert.factors?.map((factor, i) => (
                <li key={i}>{factor}</li>
              ))}
            </ul>
          </div>
        ))}
      </section>
      
      <section className="user-stats">
        <h2>User Statistics</h2>
        <table>
          <thead>
            <tr>
              <th>Username</th>
              <th>Email</th>
              <th>Role</th>
              <th>Status</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            {analytics.users.map(user => (
              <tr key={user._id}>
                <td>{user.username}</td>
                <td>{user.email}</td>
                <td>{user.role}</td>
                <td>{user.status || 'Active'}</td>
                <td>
                  <button onClick={() => editUser(user._id)}>Edit</button>
                  <button onClick={() => deleteUser(user._id)}>Delete</button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </section>
    </div>
  );
};

export default AdminAnalytics;
```

## Common Patterns

### Protected Routes with JWT

```javascript
import React from 'react';
import { Navigate } from 'react-router-dom';

const ProtectedRoute = ({ children, adminOnly = false }) => {
  const token = localStorage.getItem('token');
  
  if (!token) {
    return <Navigate to="/login" />;
  }
  
  if (adminOnly) {
    const decoded = JSON.parse(atob(token.split('.')[1]));
    if (decoded.role !== 'admin') {
      return <Navigate to="/dashboard" />;
    }
  }
  
  return children;
};

// Usage in App.js
import { BrowserRouter, Routes, Route } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={
          <ProtectedRoute>
            <UserDashboard />
          </ProtectedRoute>
        } />
        <Route path="/admin" element={
          <ProtectedRoute adminOnly={true}>
            <AdminDashboard />
          </ProtectedRoute>
        } />
      </Routes>
    </BrowserRouter>
  );
}
```

### Real-time Task Updates

```javascript
const TaskManager = () => {
  const [tasks, setTasks] = useState([]);
  
  // Polling for updates every 30 seconds
  useEffect(() => {
    const fetchTasks = async () => {
      const token = localStorage.getItem('token');
      const userId = JSON.parse(atob(token.split('.')[1])).userId;
      const res = await fetch(`http://localhost:5000/api/tasks/user/${userId}`, {
        headers: { 'Authorization': `Bearer ${token}` }
      });
      const data = await res.json();
      setTasks(data);
    };
    
    fetchTasks();
    const interval = setInterval(fetchTasks, 30000);
    return () => clearInterval(interval);
  }, []);
  
  return <div>{/* Render tasks */}</div>;
};
```

### Integrated AI Ticket Flow

```javascript
const TicketSubmission = () => {
  const [ticketData, setTicketData] = useState({
    subject: '',
    description: ''
  });
  const [aiSuggestions, setAiSuggestions] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    // First, get AI classification
    const classificationRes = await fetch('http://localhost:8000/classify-ticket', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text: `${ticketData.subject} ${ticketData.description}`,
        subject: ticketData.subject,
        description: ticketData.description
      })
    });
    const classification = await classificationRes.json();
    setAiSuggestions(classification);
    
    // Create ticket with AI suggestions
    const token = localStorage.getItem('token');
    await fetch('http://localhost:5000/api/tickets', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({
        ...ticketData,
        category: classification.category,
        priority: classification.priority,
        ai_confidence: classification.confidence
      })
    });
    
    alert('Ticket submitted successfully!');
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Subject"
        value={ticketData.subject}
        onChange={(e) => setTicketData({...ticketData, subject: e.target.value})}
      />
      <textarea
        placeholder="Description"
        value={ticketData.description}
        onChange={(e) => setTicketData({...ticketData, description: e.target.value})}
      />
      {aiSuggestions && (
        <div className="ai-suggestions">
          <p>AI Suggested Category: {aiSuggestions.category}</p>
          <p>AI Suggested Priority: {aiSuggestions.priority}</p>
          <p>Confidence: {(aiSuggestions.confidence * 100).toFixed(1)}%</p>
        </div>
      )}
      <button type="submit">Submit Ticket</button>
    </form>
  );
};
```

## Configuration

### Backend Configuration (backend/.env)

```bash
PORT=5000
MONGODB_URI=${MONGODB_URI}
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
CORS_ORIGIN=http://localhost:3000
NODE_ENV=development
LOG_LEVEL=info
```

### ML Service Configuration (ml-service/.env)

```bash
MONGODB_URI=${MONGODB_URI}
MODEL_PATH=./models
CLASSIFICATION_MODEL=ticket_classifier.pkl
RISK_MODEL=risk_predictor.pkl
ANOMALY_THRESHOLD=0.75
LOG_LEVEL=INFO
CACHE_TTL=3600
```

### Frontend Configuration (frontend/.env)

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_JWT_STORAGE_KEY=token
REACT_APP_REFRESH_INTERVAL=30000
```

## Troubleshooting

### JWT Token Expired

```javascript
// Add token refresh logic
const refreshToken = async () => {
  const token = localStorage.getItem('token');
  try {
    const response = await fetch('http://localhost:5000/api/auth/refresh', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`
      }
    });
    const data = await response.json();
    localStorage.setItem('token', data.token);
    return data.token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};
```

### MongoDB Connection Issues

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
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### ML Model Not Loading

```python
# ml-service/main.py
import os
import pickle
from pathlib import Path

def load_models():
    model_path = Path(os.getenv('MODEL_PATH', './models'))
    
    if not model_path.exists():
        model_path.mkdir(parents=True, exist_ok=True)
        print(f"Created model directory: {model_path}")
    
    models = {}
    try:
        classifier_path = model_path / 'ticket_classifier.pkl'
        if classifier_path.exists():
            with open(classifier_path, 'rb') as f:
                models['classifier'] = pickle.load(f)
        else:
            print("Classifier model not found, using default")
            models['classifier'] = None
    except Exception as e:
        print(f"Error loading models: {e}")
        models['classifier'] = None
    
    return models
```

### CORS Issues

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');

const app = express();

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### AI Service Timeout

```javascript
// Increase timeout for ML predictions
const predictWithTimeout = async (url, data, timeout = 10000) => {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeout);
  
  try {
    const response = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
      signal: controller.signal
    });
    clearTimeout(id);
    return await response.json();
  } catch (error) {
    if (error.name === 'AbortError') {
      console.error('Request timeout');
      return { error: 'Prediction service timeout' };
    }
    throw error;
  }
};
```
