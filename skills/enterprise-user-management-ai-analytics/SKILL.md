---
name: enterprise-user-management-ai-analytics
description: Full-stack enterprise user management system with AI-powered analytics for risk detection, burnout analysis, and ticket classification
triggers:
  - "set up enterprise user management system"
  - "implement AI analytics for user management"
  - "create user dashboard with task tracking"
  - "add AI-powered ticket classification"
  - "build admin dashboard with role management"
  - "integrate burnout detection for users"
  - "configure JWT authentication for enterprise app"
  - "implement kanban board with time tracking"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This is a full-stack enterprise user management system that combines user/task management with AI-powered analytics. The system provides role-based access control, task tracking with Kanban boards, support ticket management, and AI features like risk prediction, anomaly detection, burnout analysis, and intelligent ticket routing.

## Architecture

The system consists of three main components:

1. **Frontend** (React.js) - User and admin dashboards
2. **Backend** (Node.js/Express) - REST API, authentication, business logic
3. **ML Service** (FastAPI + scikit-learn) - AI analytics and predictions

## Installation

### Prerequisites

```bash
# Required
node >= 14.x
npm >= 6.x
python >= 3.8
mongodb >= 4.x
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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
JWT_SECRET=your_jwt_secret_here
JWT_EXPIRE=7d
ML_SERVICE_URL=http://localhost:8000
NODE_ENV=development
EOF

npm start
```

Backend runs at `http://localhost:5000`

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt

# Create .env file
cat > .env << EOF
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt
MODEL_PATH=./models
LOG_LEVEL=INFO
EOF

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

npm start
```

Frontend runs at `http://localhost:3000`

## Backend API Usage

### Authentication

```javascript
// Register a new user
const registerUser = async (userData) => {
  const response = await fetch('http://localhost:5000/api/auth/register', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'admin' or 'user'
    })
  });
  return response.json();
};

// Login
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
  // Store token
  localStorage.setItem('token', data.token);
  return data;
};

// Protected request helper
const apiRequest = async (endpoint, options = {}) => {
  const token = localStorage.getItem('token');
  const response = await fetch(`http://localhost:5000${endpoint}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
      ...options.headers
    }
  });
  return response.json();
};
```

### User Management (Admin)

```javascript
// Get all users
const getAllUsers = async () => {
  return await apiRequest('/api/users');
};

// Create user
const createUser = async (userData) => {
  return await apiRequest('/api/users', {
    method: 'POST',
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role,
      department: userData.department
    })
  });
};

// Update user
const updateUser = async (userId, updates) => {
  return await apiRequest(`/api/users/${userId}`, {
    method: 'PUT',
    body: JSON.stringify(updates)
  });
};

// Delete user
const deleteUser = async (userId) => {
  return await apiRequest(`/api/users/${userId}`, {
    method: 'DELETE'
  });
};
```

### Task Management

```javascript
// Create task
const createTask = async (taskData) => {
  return await apiRequest('/api/tasks', {
    method: 'POST',
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'in-progress', 'done'
    })
  });
};

// Get user tasks
const getUserTasks = async (userId) => {
  return await apiRequest(`/api/tasks/user/${userId}`);
};

// Update task status
const updateTaskStatus = async (taskId, status) => {
  return await apiRequest(`/api/tasks/${taskId}`, {
    method: 'PATCH',
    body: JSON.stringify({ status })
  });
};

// Track time on task
const trackTime = async (taskId, timeSpent) => {
  return await apiRequest(`/api/tasks/${taskId}/time`, {
    method: 'POST',
    body: JSON.stringify({ 
      timeSpent, // in minutes
      timestamp: new Date().toISOString()
    })
  });
};
```

### Support Tickets

```javascript
// Create ticket
const createTicket = async (ticketData) => {
  return await apiRequest('/api/tickets', {
    method: 'POST',
    body: JSON.stringify({
      title: ticketData.title,
      description: ticketData.description,
      priority: ticketData.priority,
      category: ticketData.category // 'technical', 'hr', 'facilities', etc.
    })
  });
};

// Get user tickets
const getUserTickets = async () => {
  return await apiRequest('/api/tickets/my-tickets');
};

// Update ticket status
const updateTicket = async (ticketId, updates) => {
  return await apiRequest(`/api/tickets/${ticketId}`, {
    method: 'PATCH',
    body: JSON.stringify(updates)
  });
};
```

## ML Service API Usage

### AI-Powered Ticket Classification

```javascript
// Classify ticket using AI
const classifyTicket = async (ticketText) => {
  const response = await fetch('http://localhost:8000/api/ml/classify-ticket', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: ticketText,
      title: 'Support Request'
    })
  });
  const result = await response.json();
  // Returns: { category: 'technical', priority: 'high', confidence: 0.89 }
  return result;
};
```

### Risk Detection

```javascript
// Analyze user risk based on behavior
const analyzeUserRisk = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/risk-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId,
      features: {
        loginAttempts: 5,
        failedLogins: 2,
        afterHoursAccess: true,
        dataDownloads: 15,
        permissionChanges: 3
      }
    })
  });
  const result = await response.json();
  // Returns: { riskScore: 0.75, riskLevel: 'high', alerts: [...] }
  return result;
};
```

### Burnout Detection

```javascript
// Detect burnout risk for user
const detectBurnout = async (userId) => {
  const response = await fetch('http://localhost:8000/api/ml/burnout-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId,
      workload: {
        tasksAssigned: 25,
        tasksCompleted: 18,
        overtimeHours: 15,
        weekendWork: 8,
        averageTaskCompletionTime: 120 // minutes
      }
    })
  });
  const result = await response.json();
  // Returns: { burnoutRisk: 0.82, recommendation: 'Reduce workload', factors: [...] }
  return result;
};
```

### Anomaly Detection

```javascript
// Detect anomalies in user behavior
const detectAnomalies = async (userActivity) => {
  const response = await fetch('http://localhost:8000/api/ml/anomaly-detection', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      activities: userActivity.map(activity => ({
        timestamp: activity.timestamp,
        action: activity.action,
        ip: activity.ip,
        location: activity.location,
        deviceId: activity.deviceId
      }))
    })
  });
  const result = await response.json();
  // Returns: { anomalies: [...], anomalyScore: 0.65 }
  return result;
};
```

### Predictive Project Insights

```javascript
// Predict project delays
const predictProjectDelay = async (projectData) => {
  const response = await fetch('http://localhost:8000/api/ml/predict-delay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      projectId: projectData.id,
      totalTasks: projectData.totalTasks,
      completedTasks: projectData.completedTasks,
      daysRemaining: projectData.daysRemaining,
      teamSize: projectData.teamSize,
      complexityScore: projectData.complexityScore
    })
  });
  const result = await response.json();
  // Returns: { delayProbability: 0.68, estimatedDelay: 5, recommendations: [...] }
  return result;
};
```

## React Frontend Patterns

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';
import { apiRequest } from '../utils/api';

const UserDashboard = () => {
  const [tasks, setTasks] = useState({ todo: [], inProgress: [], done: [] });
  const [tickets, setTickets] = useState([]);
  const [burnoutRisk, setBurnoutRisk] = useState(null);

  useEffect(() => {
    loadUserData();
  }, []);

  const loadUserData = async () => {
    try {
      // Load tasks
      const userTasks = await apiRequest('/api/tasks/my-tasks');
      const categorized = {
        todo: userTasks.filter(t => t.status === 'todo'),
        inProgress: userTasks.filter(t => t.status === 'in-progress'),
        done: userTasks.filter(t => t.status === 'done')
      };
      setTasks(categorized);

      // Load tickets
      const userTickets = await apiRequest('/api/tickets/my-tickets');
      setTickets(userTickets);

      // Check burnout risk
      const burnout = await fetch('http://localhost:8000/api/ml/burnout-detection', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          userId: localStorage.getItem('userId'),
          workload: {
            tasksAssigned: userTasks.length,
            tasksCompleted: categorized.done.length,
            overtimeHours: 0,
            weekendWork: 0,
            averageTaskCompletionTime: 90
          }
        })
      }).then(r => r.json());
      setBurnoutRisk(burnout);
    } catch (error) {
      console.error('Error loading dashboard:', error);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    await apiRequest(`/api/tasks/${taskId}`, {
      method: 'PATCH',
      body: JSON.stringify({ status: newStatus })
    });
    loadUserData();
  };

  return (
    <div className="dashboard">
      {burnoutRisk && burnoutRisk.burnoutRisk > 0.7 && (
        <div className="alert alert-warning">
          ⚠️ High burnout risk detected: {burnoutRisk.recommendation}
        </div>
      )}
      
      <div className="kanban-board">
        <TaskColumn 
          title="To Do" 
          tasks={tasks.todo} 
          onMove={(id) => moveTask(id, 'in-progress')} 
        />
        <TaskColumn 
          title="In Progress" 
          tasks={tasks.inProgress} 
          onMove={(id) => moveTask(id, 'done')} 
        />
        <TaskColumn 
          title="Done" 
          tasks={tasks.done} 
        />
      </div>

      <TicketList tickets={tickets} />
    </div>
  );
};

export default UserDashboard;
```

### Admin Analytics Component

```javascript
import React, { useState, useEffect } from 'react';
import { apiRequest } from '../utils/api';

const AdminAnalytics = () => {
  const [users, setUsers] = useState([]);
  const [riskUsers, setRiskUsers] = useState([]);
  const [analytics, setAnalytics] = useState(null);

  useEffect(() => {
    loadAnalytics();
  }, []);

  const loadAnalytics = async () => {
    // Get all users
    const allUsers = await apiRequest('/api/users');
    setUsers(allUsers);

    // Analyze risk for each user
    const riskyUsers = [];
    for (const user of allUsers) {
      const riskAnalysis = await fetch('http://localhost:8000/api/ml/risk-detection', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          userId: user._id,
          features: user.securityMetrics || {}
        })
      }).then(r => r.json());

      if (riskAnalysis.riskLevel === 'high') {
        riskyUsers.push({ ...user, risk: riskAnalysis });
      }
    }
    setRiskUsers(riskyUsers);

    // Get overall analytics
    const stats = await apiRequest('/api/analytics/overview');
    setAnalytics(stats);
  };

  return (
    <div className="admin-analytics">
      <h2>Organization Analytics</h2>
      
      {analytics && (
        <div className="stats-grid">
          <StatCard title="Total Users" value={analytics.totalUsers} />
          <StatCard title="Active Tasks" value={analytics.activeTasks} />
          <StatCard title="Open Tickets" value={analytics.openTickets} />
          <StatCard title="Completion Rate" value={`${analytics.completionRate}%`} />
        </div>
      )}

      <h3>High Risk Users</h3>
      <table className="risk-table">
        <thead>
          <tr>
            <th>User</th>
            <th>Risk Score</th>
            <th>Alerts</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {riskUsers.map(user => (
            <tr key={user._id}>
              <td>{user.name}</td>
              <td>{(user.risk.riskScore * 100).toFixed(0)}%</td>
              <td>{user.risk.alerts.join(', ')}</td>
              <td>
                <button onClick={() => reviewUser(user._id)}>Review</button>
              </td>
            </tr>
          ))}
        </tbody>
      </table>
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
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# JWT
JWT_SECRET=your_strong_secret_key
JWT_EXPIRE=7d

# ML Service
ML_SERVICE_URL=http://localhost:8000

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=${SMTP_USER}
SMTP_PASS=${SMTP_PASS}

# Rate Limiting
RATE_LIMIT_WINDOW=15
RATE_LIMIT_MAX=100
```

### ML Service Configuration (ml-service/.env)

```bash
# Database
MONGODB_URI=mongodb://localhost:27017/enterprise_user_mgmt

# Model Settings
MODEL_PATH=./models
MODEL_RETRAIN_INTERVAL=86400

# Logging
LOG_LEVEL=INFO
LOG_FILE=./logs/ml-service.log

# Thresholds
RISK_THRESHOLD=0.7
BURNOUT_THRESHOLD=0.75
ANOMALY_THRESHOLD=0.8
```

### Frontend Configuration (frontend/.env)

```bash
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
REACT_APP_ENABLE_AI=true
```

## Common Patterns

### Implementing Time Tracking

```javascript
// Time tracking hook
import { useState, useEffect } from 'react';

const useTaskTimer = (taskId) => {
  const [seconds, setSeconds] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setSeconds(s => s + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const start = () => setIsRunning(true);
  const pause = () => setIsRunning(false);
  const reset = () => {
    setSeconds(0);
    setIsRunning(false);
  };
  
  const save = async () => {
    if (seconds > 0) {
      await apiRequest(`/api/tasks/${taskId}/time`, {
        method: 'POST',
        body: JSON.stringify({ 
          timeSpent: Math.floor(seconds / 60),
          timestamp: new Date().toISOString()
        })
      });
      reset();
    }
  };

  return { seconds, isRunning, start, pause, reset, save };
};

// Usage in component
const TaskTimer = ({ taskId }) => {
  const { seconds, isRunning, start, pause, save } = useTaskTimer(taskId);
  
  const formatTime = (s) => {
    const hrs = Math.floor(s / 3600);
    const mins = Math.floor((s % 3600) / 60);
    const secs = s % 60;
    return `${hrs}:${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  return (
    <div className="timer">
      <div className="time-display">{formatTime(seconds)}</div>
      <button onClick={isRunning ? pause : start}>
        {isRunning ? 'Pause' : 'Start'}
      </button>
      <button onClick={save}>Save Time</button>
    </div>
  );
};
```

### Creating AI-Enhanced Ticket Form

```javascript
const SmartTicketForm = () => {
  const [formData, setFormData] = useState({ title: '', description: '' });
  const [aiSuggestions, setAiSuggestions] = useState(null);

  const handleDescriptionChange = async (e) => {
    const description = e.target.value;
    setFormData({ ...formData, description });

    // Auto-classify as user types
    if (description.length > 50) {
      const classification = await fetch('http://localhost:8000/api/ml/classify-ticket', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          text: description,
          title: formData.title
        })
      }).then(r => r.json());
      
      setAiSuggestions(classification);
    }
  };

  const submitTicket = async (e) => {
    e.preventDefault();
    
    await apiRequest('/api/tickets', {
      method: 'POST',
      body: JSON.stringify({
        ...formData,
        category: aiSuggestions?.category || 'general',
        priority: aiSuggestions?.priority || 'medium'
      })
    });
  };

  return (
    <form onSubmit={submitTicket}>
      <input 
        type="text"
        placeholder="Title"
        value={formData.title}
        onChange={(e) => setFormData({ ...formData, title: e.target.value })}
      />
      
      <textarea
        placeholder="Description"
        value={formData.description}
        onChange={handleDescriptionChange}
      />

      {aiSuggestions && (
        <div className="ai-suggestions">
          <p>AI Suggested Category: <strong>{aiSuggestions.category}</strong></p>
          <p>AI Suggested Priority: <strong>{aiSuggestions.priority}</strong></p>
          <p>Confidence: {(aiSuggestions.confidence * 100).toFixed(0)}%</p>
        </div>
      )}

      <button type="submit">Create Ticket</button>
    </form>
  );
};
```

## Troubleshooting

### MongoDB Connection Issues

```bash
# Check if MongoDB is running
sudo systemctl status mongod

# Start MongoDB
sudo systemctl start mongod

# Verify connection
mongo --eval "db.adminCommand('ping')"
```

### JWT Token Expiration

```javascript
// Add token refresh logic
const refreshToken = async () => {
  try {
    const response = await fetch('http://localhost:5000/api/auth/refresh', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    });
    const { token } = await response.json();
    localStorage.setItem('token', token);
    return token;
  } catch (error) {
    // Redirect to login
    window.location.href = '/login';
  }
};

// Add interceptor to refresh expired tokens
const apiRequest = async (endpoint, options = {}) => {
  let token = localStorage.getItem('token');
  let response = await fetch(`http://localhost:5000${endpoint}`, {
    ...options,
    headers: {
      'Authorization': `Bearer ${token}`,
      ...options.headers
    }
  });

  if (response.status === 401) {
    token = await refreshToken();
    response = await fetch(`http://localhost:5000${endpoint}`, {
      ...options,
      headers: {
        'Authorization': `Bearer ${token}`,
        ...options.headers
      }
    });
  }

  return response.json();
};
```

### ML Model Not Loading

```bash
# Check if models directory exists
ls -la ml-service/models/

# Retrain models if missing
cd ml-service
python scripts/train_models.py

# Check ML service logs
tail -f logs/ml-service.log
```

### CORS Issues

```javascript
// In backend/server.js, ensure CORS is configured
const cors = require('cors');

app.use(cors({
  origin: process.env.FRONTEND_URL || 'http://localhost:3000',
  credentials: true
}));
```

### Performance Issues with Large Datasets

```javascript
// Implement pagination for user lists
const getUsersPaginated = async (page = 1, limit = 20) => {
  return await apiRequest(`/api/users?page=${page}&limit=${limit}`);
};

// Use debouncing for search
import { debounce } from 'lodash';

const searchUsers = debounce(async (query) => {
  return await apiRequest(`/api/users/search?q=${query}`);
}, 300);
```

This skill provides comprehensive coverage of the Enterprise User Management System with AI Analytics, enabling AI agents to help developers implement user management, task tracking, and AI-powered analytics features effectively.
