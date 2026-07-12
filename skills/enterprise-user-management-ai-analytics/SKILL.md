---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, anomaly analysis, and predictive insights
triggers:
  - "set up enterprise user management system"
  - "integrate AI analytics for user management"
  - "implement JWT authentication with user roles"
  - "create kanban board with task tracking"
  - "add AI-based ticket classification"
  - "build admin dashboard with analytics"
  - "configure ML service for anomaly detection"
  - "implement burnout prediction system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

A full-stack web application for enterprise user and task management with integrated AI analytics. The system provides role-based access control, task management with Kanban boards, support ticket handling, and AI-powered insights including risk detection, anomaly detection, burnout analysis, and predictive project analytics.

## Architecture Overview

The system consists of three main components:

- **Frontend**: React.js application for user and admin interfaces
- **Backend**: Node.js/Express REST API with MongoDB
- **ML Service**: FastAPI-based machine learning service for AI analytics

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

# Start backend
npm start
```

Backend runs at `http://localhost:5000`

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

# Start ML service
uvicorn main:app --reload --port 8000
```

ML service runs at `http://localhost:8000`

### Frontend Setup

```bash
cd frontend
npm install

# Create .env file
cat > .env << EOF
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
EOF

# Start frontend
npm start
```

Frontend runs at `http://localhost:3000`

## Backend API Reference

### Authentication Endpoints

```javascript
// User registration
const register = async (userData) => {
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

// User login
const login = async (credentials) => {
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

// Get authenticated user profile
const getProfile = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/auth/profile', {
    headers: {
      'Authorization': `Bearer ${token}`
    }
  });
  return response.json();
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Create new user
const createUser = async (userData) => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/users', {
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

// Delete user
const deleteUser = async (userId) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/users/${userId}`, {
    method: 'DELETE',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};
```

### Task Management

```javascript
// Get all tasks
const getTasks = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tasks', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Create task
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
      priority: taskData.priority || 'medium',
      status: taskData.status || 'todo',
      dueDate: taskData.dueDate
    })
  });
  return response.json();
};

// Update task status (Kanban)
const updateTaskStatus = async (taskId, newStatus) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}`, {
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
const trackTime = async (taskId, timeSpent) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ timeSpent })
  });
  return response.json();
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
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category
    })
  });
  return response.json();
};

// Get user tickets
const getMyTickets = async () => {
  const token = localStorage.getItem('token');
  const response = await fetch('http://localhost:5000/api/tickets/my', {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  return response.json();
};

// Update ticket status (Admin)
const updateTicketStatus = async (ticketId, status) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000/api/tickets/${ticketId}`, {
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

### AI Analytics Endpoints

```javascript
// Risk prediction
const predictRisk = async (userData) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-prediction', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: userData.userId,
      taskCount: userData.taskCount,
      avgTaskDuration: userData.avgTaskDuration,
      ticketCount: userData.ticketCount,
      loginFrequency: userData.loginFrequency
    })
  });
  return response.json();
  // Returns: { riskScore: 0.75, riskLevel: "high", factors: [...] }
};

// Anomaly detection
const detectAnomaly = async (activityData) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: activityData.userId,
      loginTime: activityData.loginTime,
      ipAddress: activityData.ipAddress,
      activityPattern: activityData.activityPattern
    })
  });
  return response.json();
  // Returns: { isAnomaly: true, anomalyScore: 0.85, type: "unusual_login_time" }
};

// Burnout detection
const detectBurnout = async (userId) => {
  const response = await fetch(`http://localhost:8000/api/ml/burnout-detection/${userId}`, {
    method: 'GET'
  });
  return response.json();
  // Returns: { burnoutRisk: 0.65, level: "moderate", recommendations: [...] }
};

// Ticket classification
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/ticket-classification', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticketText.title,
      description: ticketText.description
    })
  });
  return response.json();
  // Returns: { category: "technical", priority: "high", suggestedAssignee: "team_a" }
};

// Predictive project insights
const getPredictiveInsights = async (projectId) => {
  const response = await fetch(`http://localhost:8000/api/ml/project-insights/${projectId}`, {
    method: 'GET'
  });
  return response.json();
  // Returns: { delayProbability: 0.45, estimatedCompletion: "2026-06-15", bottlenecks: [...] }
};
```

## Frontend Component Patterns

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [tickets, setTickets] = useState([]);
  const [burnoutRisk, setBurnoutRisk] = useState(null);

  useEffect(() => {
    fetchUserData();
  }, []);

  const fetchUserData = async () => {
    const token = localStorage.getItem('token');
    
    // Fetch tasks
    const tasksRes = await fetch('http://localhost:5000/api/tasks', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const tasksData = await tasksRes.json();
    setTasks(tasksData);

    // Fetch tickets
    const ticketsRes = await fetch('http://localhost:5000/api/tickets/my', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const ticketsData = await ticketsRes.json();
    setTickets(ticketsData);

    // Get burnout risk
    const userId = JSON.parse(atob(token.split('.')[1])).userId;
    const burnoutRes = await fetch(`http://localhost:8000/api/ml/burnout-detection/${userId}`);
    const burnoutData = await burnoutRes.json();
    setBurnoutRisk(burnoutData);
  };

  return (
    <div className="dashboard">
      <h1>My Dashboard</h1>
      
      {burnoutRisk && burnoutRisk.level === 'high' && (
        <div className="alert alert-warning">
          ⚠️ High burnout risk detected. Consider taking a break.
        </div>
      )}

      <div className="stats-grid">
        <div className="stat-card">
          <h3>Active Tasks</h3>
          <p>{tasks.filter(t => t.status !== 'done').length}</p>
        </div>
        <div className="stat-card">
          <h3>Open Tickets</h3>
          <p>{tickets.filter(t => t.status === 'open').length}</p>
        </div>
      </div>

      <TaskList tasks={tasks} onUpdate={fetchUserData} />
    </div>
  );
};

export default UserDashboard;
```

### Kanban Board Component

```javascript
import React from 'react';
import { DragDropContext, Droppable, Draggable } from 'react-beautiful-dnd';

const KanbanBoard = ({ tasks, onTaskMove }) => {
  const columns = {
    todo: tasks.filter(t => t.status === 'todo'),
    inProgress: tasks.filter(t => t.status === 'in_progress'),
    done: tasks.filter(t => t.status === 'done')
  };

  const handleDragEnd = async (result) => {
    if (!result.destination) return;

    const { draggableId, destination } = result;
    const newStatus = destination.droppableId;

    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${draggableId}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });

    onTaskMove();
  };

  return (
    <DragDropContext onDragEnd={handleDragEnd}>
      <div className="kanban-board">
        {Object.entries(columns).map(([columnId, columnTasks]) => (
          <Droppable key={columnId} droppableId={columnId}>
            {(provided) => (
              <div
                className="kanban-column"
                ref={provided.innerRef}
                {...provided.droppableProps}
              >
                <h3>{columnId.replace('_', ' ').toUpperCase()}</h3>
                {columnTasks.map((task, index) => (
                  <Draggable key={task._id} draggableId={task._id} index={index}>
                    {(provided) => (
                      <div
                        className="task-card"
                        ref={provided.innerRef}
                        {...provided.draggableProps}
                        {...provided.dragHandleProps}
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

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const AdminAnalytics = () => {
  const [analytics, setAnalytics] = useState(null);
  const [highRiskUsers, setHighRiskUsers] = useState([]);

  useEffect(() => {
    fetchAnalytics();
  }, []);

  const fetchAnalytics = async () => {
    const token = localStorage.getItem('token');

    // Fetch organization analytics
    const analyticsRes = await fetch('http://localhost:5000/api/admin/analytics', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const analyticsData = await analyticsRes.json();
    setAnalytics(analyticsData);

    // Fetch high-risk users
    const usersRes = await fetch('http://localhost:5000/api/users', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    const users = await usersRes.json();

    // Get risk predictions for each user
    const riskPromises = users.map(async (user) => {
      const riskRes = await fetch('http://localhost:8000/api/ml/risk-prediction', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          userId: user._id,
          taskCount: user.taskCount || 0,
          avgTaskDuration: user.avgTaskDuration || 0,
          ticketCount: user.ticketCount || 0,
          loginFrequency: user.loginFrequency || 0
        })
      });
      const riskData = await riskRes.json();
      return { ...user, riskScore: riskData.riskScore };
    });

    const usersWithRisk = await Promise.all(riskPromises);
    setHighRiskUsers(usersWithRisk.filter(u => u.riskScore > 0.7));
  };

  return (
    <div className="admin-analytics">
      <h1>Analytics Dashboard</h1>

      {analytics && (
        <div className="analytics-grid">
          <div className="metric-card">
            <h3>Total Users</h3>
            <p>{analytics.totalUsers}</p>
          </div>
          <div className="metric-card">
            <h3>Active Tasks</h3>
            <p>{analytics.activeTasks}</p>
          </div>
          <div className="metric-card">
            <h3>Open Tickets</h3>
            <p>{analytics.openTickets}</p>
          </div>
          <div className="metric-card">
            <h3>Avg Completion Time</h3>
            <p>{analytics.avgCompletionTime}h</p>
          </div>
        </div>
      )}

      <div className="high-risk-section">
        <h2>High Risk Users</h2>
        {highRiskUsers.map(user => (
          <div key={user._id} className="risk-card">
            <h4>{user.username}</h4>
            <p>Risk Score: {(user.riskScore * 100).toFixed(0)}%</p>
            <button onClick={() => viewUserDetails(user._id)}>
              View Details
            </button>
          </div>
        ))}
      </div>
    </div>
  );
};

export default AdminAnalytics;
```

## Configuration

### Backend Configuration (backend/.env)

```bash
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=${MONGODB_URI}

# JWT
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# CORS
CORS_ORIGIN=http://localhost:3000

# Logging
LOG_LEVEL=info
```

### ML Service Configuration (ml-service/.env)

```bash
# Database
MONGODB_URI=${MONGODB_URI}

# Model Configuration
MODEL_PATH=./models
MODEL_UPDATE_INTERVAL=3600

# Logging
LOG_LEVEL=INFO

# Feature Flags
ENABLE_ANOMALY_DETECTION=true
ENABLE_BURNOUT_PREDICTION=true
ENABLE_RISK_SCORING=true
```

### Frontend Configuration (frontend/.env)

```bash
# API URLs
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000

# Feature Flags
REACT_APP_ENABLE_AI_FEATURES=true
REACT_APP_ENABLE_NOTIFICATIONS=true
```

## Common Patterns

### Implementing Role-Based Access Control

```javascript
// Middleware for role checking
const checkRole = (allowedRoles) => {
  return (req, res, next) => {
    const userRole = req.user.role; // Set by JWT middleware
    
    if (!allowedRoles.includes(userRole)) {
      return res.status(403).json({ 
        error: 'Access denied. Insufficient permissions.' 
      });
    }
    
    next();
  };
};

// Usage in routes
app.get('/api/admin/users', 
  authenticateJWT, 
  checkRole(['admin']), 
  getAllUsers
);

app.post('/api/tasks', 
  authenticateJWT, 
  checkRole(['admin', 'manager']), 
  createTask
);
```

### Time Tracking Implementation

```javascript
const TimeTracker = () => {
  const [isTracking, setIsTracking] = useState(false);
  const [startTime, setStartTime] = useState(null);
  const [elapsedTime, setElapsedTime] = useState(0);

  useEffect(() => {
    let interval;
    if (isTracking) {
      interval = setInterval(() => {
        setElapsedTime(Date.now() - startTime);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isTracking, startTime]);

  const startTracking = () => {
    setStartTime(Date.now());
    setIsTracking(true);
  };

  const stopTracking = async (taskId) => {
    setIsTracking(false);
    const timeSpent = Math.floor(elapsedTime / 1000 / 60); // minutes

    const token = localStorage.getItem('token');
    await fetch(`http://localhost:5000/api/tasks/${taskId}/time`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({ timeSpent })
    });

    setElapsedTime(0);
  };

  return (
    <div className="time-tracker">
      <p>{new Date(elapsedTime).toISOString().substr(11, 8)}</p>
      {!isTracking ? (
        <button onClick={startTracking}>Start</button>
      ) : (
        <button onClick={() => stopTracking(currentTaskId)}>Stop</button>
      )}
    </div>
  );
};
```

### AI-Powered Ticket Auto-Assignment

```javascript
const autoAssignTicket = async (ticket) => {
  // Classify ticket using ML service
  const classificationRes = await fetch('http://localhost:8000/api/ml/ticket-classification', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      title: ticket.title,
      description: ticket.description
    })
  });
  const classification = await classificationRes.json();

  // Assign to suggested team/user
  const token = localStorage.getItem('token');
  await fetch(`http://localhost:5000/api/tickets/${ticket._id}`, {
    method: 'PATCH',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      assignedTo: classification.suggestedAssignee,
      category: classification.category,
      priority: classification.priority
    })
  });

  return classification;
};
```

## Troubleshooting

### JWT Token Expired

```javascript
// Implement token refresh logic
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  const response = await fetch('http://localhost:5000/api/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  const data = await response.json();
  localStorage.setItem('token', data.token);
  return data.token;
};

// Axios interceptor for auto-refresh
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

### MongoDB Connection Issues

```javascript
// Connection retry logic
const connectDB = async (retries = 5) => {
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
      console.log(`Connection attempt ${i + 1} failed. Retrying...`);
      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }
  throw new Error('Failed to connect to MongoDB after multiple attempts');
};
```

### ML Model Not Loading

```python
# ml-service/main.py
import os
import pickle
from pathlib import Path

def load_models():
    model_path = Path(os.getenv('MODEL_PATH', './models'))
    models = {}
    
    try:
        if (model_path / 'risk_model.pkl').exists():
            with open(model_path / 'risk_model.pkl', 'rb') as f:
                models['risk'] = pickle.load(f)
        else:
            # Initialize default model
            from sklearn.ensemble import RandomForestClassifier
            models['risk'] = RandomForestClassifier()
            print("Warning: Using default risk model")
        
        return models
    except Exception as e:
        print(f"Error loading models: {e}")
        return {}
```

### CORS Issues

```javascript
// backend/server.js
const cors = require('cors');

app.use(cors({
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

### Performance Optimization for Large Datasets

```javascript
// Implement pagination
const getTasks = async (page = 1, limit = 20) => {
  const token = localStorage.getItem('token');
  const response = await fetch(
    `http://localhost:5000/api/tasks?page=${page}&limit=${limit}`,
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
  return response.json();
};

// Backend implementation
app.get('/api/tasks', authenticateJWT, async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const skip = (page - 1) * limit;

  const tasks = await Task.find()
    .skip(skip)
    .limit(limit)
    .sort({ createdAt: -1 });

  const total = await Task.countDocuments();

  res.json({
    tasks,
    currentPage: page,
    totalPages: Math.ceil(total / limit),
    totalTasks: total
  });
});
```
