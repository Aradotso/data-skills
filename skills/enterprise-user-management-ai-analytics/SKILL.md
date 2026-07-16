---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics for task tracking, ticket classification, risk prediction, and burnout detection
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create task management with AI insights"
  - "build user management system with risk detection"
  - "add AI ticket classification and routing"
  - "integrate burnout prediction for users"
  - "deploy user management with ML analytics"
  - "configure AI-driven task tracking system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is a full-stack enterprise user management system that combines user administration, task tracking, and support ticket management with AI-powered analytics. It features JWT authentication, role-based access control, Kanban task boards, and ML capabilities including risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## Project Architecture

The system consists of three main components:

- **Frontend**: React.js application for user and admin interfaces
- **Backend**: Node.js/Express REST API with MongoDB
- **ML Service**: FastAPI-based machine learning service for AI analytics

## Installation & Setup

### Prerequisites

```bash
# Required: Node.js 14+, Python 3.8+, MongoDB
node --version
python --version
mongod --version
```

### Complete Setup

```bash
# Clone repository
git clone https://github.com/Nareshkumar2583/Enterprise-User-Management-System-with-AI-Analytics.git
cd Enterprise-User-Management-System-with-AI-Analytics

# Backend setup
cd backend
npm install
# Create .env file
cat > .env << EOF
PORT=5000
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=${JWT_SECRET}
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF
npm start

# ML Service setup (new terminal)
cd ml-service
pip install -r requirements.txt
# Create .env file
cat > .env << EOF
MODEL_PATH=./models
DB_CONNECTION=${MONGODB_URI}
LOG_LEVEL=INFO
EOF
uvicorn main:app --reload --port 8000

# Frontend setup (new terminal)
cd frontend
npm install
# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF
npm start
```

## Backend API Usage

### Authentication

```javascript
// Login endpoint
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

// Protected API request
const fetchUserProfile = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users/profile', {
    headers: { 
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  return await response.json();
};
```

### User Management (Admin)

```javascript
// Create new user
const createUser = async (userData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/admin/users', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      role: userData.role, // 'admin' or 'user'
      department: userData.department
    })
  });
  return await response.json();
};

// Update user
const updateUser = async (userId, updates) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/admin/users/${userId}`, {
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
  const response = await fetch(`http://localhost:5000/api/admin/users/${userId}`, {
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
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in-progress', 'done'
    })
  });
  return await response.json();
};

// Update task status (Kanban)
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
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
  const response = await fetch('http://localhost:5000/api/tasks/my-tasks', {
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
  const response = await fetch('http://localhost:5000/api/tickets', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category // 'technical', 'account', 'general'
    })
  });
  return await response.json();
};

// Get tickets
const getTickets = async (filters = {}) => {
  const token = localStorage.getItem('token');
  const queryParams = new URLSearchParams(filters).toString();
  const response = await fetch(`http://localhost:5000/api/tickets?${queryParams}`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return await response.json();
};
```

## ML Service Integration

### AI Ticket Classification

```javascript
// Classify ticket automatically
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/ai/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      subject: "Issue with login"
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.92 }
  return result;
};
```

### Risk Prediction

```javascript
// Predict user risk level
const predictUserRisk = async (userId, userMetrics) => {
  const response = await fetch('http://localhost:8000/ai/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      failedLogins: userMetrics.failedLogins,
      unusualActivity: userMetrics.unusualActivity,
      workloadScore: userMetrics.workloadScore,
      taskCompletionRate: userMetrics.taskCompletionRate
    })
  });
  const result = await response.json();
  // Returns: { riskLevel: 'high', score: 0.78, factors: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// Analyze user burnout risk
const detectBurnout = async (userId, workData) => {
  const response = await fetch('http://localhost:8000/ai/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      hoursWorked: workData.weeklyHours,
      tasksAssigned: workData.taskCount,
      tasksCompleted: workData.completedCount,
      overtimeHours: workData.overtime,
      consecutiveDays: workData.workStreak
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 'medium', score: 0.65, recommendations: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalous user behavior
const detectAnomalies = async (userId, activityLog) => {
  const response = await fetch('http://localhost:8000/ai/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userId,
      activities: activityLog.map(log => ({
        timestamp: log.timestamp,
        action: log.action,
        ipAddress: log.ip,
        location: log.location,
        deviceId: log.device
      }))
    })
  });
  const result = await response.json();
  // Returns: { isAnomalous: true, anomalyScore: 0.85, details: [...] }
  return result;
};
```

### Predictive Project Insights

```javascript
// Predict project completion
const predictProjectDelay = async (projectId, projectData) => {
  const response = await fetch('http://localhost:8000/ai/project-insights', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectId,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      daysRemaining: projectData.daysToDeadline,
      teamSize: projectData.teamSize,
      avgCompletionRate: projectData.dailyCompletionRate
    })
  });
  const result = await response.json();
  // Returns: { delayPrediction: true, estimatedDelay: 5, confidence: 0.82 }
  return result;
};
```

## React Frontend Patterns

### Protected Routes

```javascript
// ProtectedRoute.js
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

export default ProtectedRoute;
```

### Task Board Component

```javascript
// TaskBoard.js
import React, { useState, useEffect } from 'react';

const TaskBoard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });

  useEffect(() => {
    fetchTasks();
  }, []);

  const fetchTasks = async () => {
    const token = localStorage.getItem('token');
    const response = await fetch('http://localhost:5000/api/tasks/my-tasks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await response.json();
    
    setTasks({
      todo: data.filter(t => t.status === 'todo'),
      inProgress: data.filter(t => t.status === 'in-progress'),
      done: data.filter(t => t.status === 'done')
    });
  };

  const moveTask = async (taskId, newStatus) => {
    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
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
      <div className="column">
        <h3>To Do</h3>
        {tasks.todo.map(task => (
          <div key={task._id} className="task-card">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
            <button onClick={() => moveTask(task._id, 'in-progress')}>
              Start
            </button>
          </div>
        ))}
      </div>
      <div className="column">
        <h3>In Progress</h3>
        {tasks.inProgress.map(task => (
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
        <h3>Done</h3>
        {tasks.done.map(task => (
          <div key={task._id} className="task-card completed">
            <h4>{task.title}</h4>
            <p>{task.description}</p>
          </div>
        ))}
      </div>
    </div>
  );
};

export default TaskBoard;
```

### AI Analytics Dashboard

```javascript
// AdminAnalytics.js
import React, { useState, useEffect } from 'react';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState({
    highRiskUsers: [],
    burnoutAlerts: [],
    anomalies: []
  });

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch high-risk users
    const usersResponse = await fetch('http://localhost:5000/api/admin/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const users = await usersResponse.json();
    
    // Analyze each user with AI
    const riskPromises = users.map(async (user) => {
      const metrics = await fetch(`http://localhost:5000/api/admin/users/${user._id}/metrics`, {
        headers: { 'Authorization': `Bearer ${token}` }
      }).then(r => r.json());
      
      const risk = await fetch('http://localhost:8000/ai/risk-prediction', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ userId: user._id, ...metrics })
      }).then(r => r.json());
      
      return { ...user, risk };
    });
    
    const analyzedUsers = await Promise.all(riskPromises);
    const highRisk = analyzedUsers.filter(u => u.risk.riskLevel === 'high');
    
    setAnalytics(prev => ({ ...prev, highRiskUsers: highRisk }));
  };

  return (
    <div className="analytics-dashboard">
      <h2>AI Analytics Dashboard</h2>
      
      <section className="risk-alerts">
        <h3>High Risk Users</h3>
        {analytics.highRiskUsers.map(user => (
          <div key={user._id} className="alert-card">
            <h4>{user.name}</h4>
            <p>Risk Score: {(user.risk.score * 100).toFixed(0)}%</p>
            <ul>
              {user.risk.factors.map((factor, i) => (
                <li key={i}>{factor}</li>
              ))}
            </ul>
          </div>
        ))}
      </section>
    </div>
  );
};

export default AdminAnalytics;
```

## Configuration

### Backend Configuration

```javascript
// backend/config/database.js
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB connected');
  } catch (error) {
    console.error('MongoDB connection error:', error);
    process.exit(1);
  }
};

module.exports = connectDB;
```

### JWT Middleware

```javascript
// backend/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization')?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No authentication token' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = decoded.userId;
    req.userRole = decoded.role;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

const adminOnly = (req, res, next) => {
  if (req.userRole !== 'admin') {
    return res.status(403).json({ error: 'Admin access required' });
  }
  next();
};

module.exports = { authMiddleware, adminOnly };
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Check connection string
echo $MONGODB_URI
```

### JWT Token Errors

```javascript
// Clear expired tokens
localStorage.removeItem('token');
localStorage.removeItem('userRole');

// Verify token format in requests
const token = localStorage.getItem('token');
console.log('Token:', token ? 'Present' : 'Missing');
```

### ML Service Not Responding

```bash
# Check ML service logs
cd ml-service
tail -f logs/ml_service.log

# Verify Python dependencies
pip list | grep -E "(fastapi|scikit-learn|river)"

# Test ML service directly
curl -X POST http://localhost:8000/ai/classify-ticket \
  -H "Content-Type: application/json" \
  -d '{"text":"Cannot login","subject":"Login issue"}'
```

### CORS Issues

```javascript
// backend/index.js
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Task Status Not Updating

```javascript
// Ensure status values match exactly
const validStatuses = ['todo', 'in-progress', 'done'];

const updateTaskStatus = async (taskId, status) => {
  if (!validStatuses.includes(status)) {
    console.error('Invalid status:', status);
    return;
  }
  // ... rest of update logic
};
```

This skill enables AI agents to help developers implement and configure the Enterprise User Management System with its full suite of AI-powered analytics capabilities.
