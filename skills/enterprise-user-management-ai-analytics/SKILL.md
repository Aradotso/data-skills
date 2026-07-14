---
name: enterprise-user-management-ai-analytics
description: Full-stack user management system with AI-powered analytics, risk detection, burnout analysis, and task management for enterprise organizations
triggers:
  - "set up enterprise user management system"
  - "implement AI-powered user analytics"
  - "create user management with risk detection"
  - "build task management with burnout analysis"
  - "integrate AI analytics for user management"
  - "develop enterprise user dashboard with ML"
  - "add anomaly detection to user system"
  - "configure AI ticket classification system"
---

# Enterprise User Management System with AI Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Enterprise User Management System with AI Analytics is a full-stack JavaScript application that combines user management, task tracking, and support tickets with AI-powered insights. The system provides risk detection, anomaly detection, burnout analysis, and predictive project insights to improve organizational efficiency.

**Architecture:**
- Frontend: React.js
- Backend: Node.js with Express
- ML Service: FastAPI (Python)
- Database: MongoDB
- Authentication: JWT

## Installation

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

```env
PORT=5000
MONGODB_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
JWT_EXPIRE=7d
NODE_ENV=development
ML_SERVICE_URL=http://localhost:8000
```

Start backend:

```bash
npm start
# Development mode with auto-reload
npm run dev
```

### ML Service Setup

```bash
cd ml-service
pip install -r requirements.txt
```

Create `.env` file in ml-service directory:

```env
MODEL_PATH=./models
LOG_LEVEL=INFO
BACKEND_URL=http://localhost:5000
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

Create `.env` file in frontend directory:

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_ML_API_URL=http://localhost:8000
```

Start frontend:

```bash
npm start
```

## Backend API Structure

### Authentication Endpoints

```javascript
// POST /api/auth/register
const registerUser = async (userData) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/auth/register`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      name: userData.name,
      email: userData.email,
      password: userData.password,
      role: userData.role // 'user' or 'admin'
    })
  });
  return await response.json();
};

// POST /api/auth/login
const loginUser = async (credentials) => {
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
  // Store token for subsequent requests
  localStorage.setItem('token', data.token);
  return data;
};
```

### User Management Endpoints

```javascript
// GET /api/users - Get all users (Admin only)
const getAllUsers = async (token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/users`, {
    headers: {
      'Authorization': `Bearer ${token}`,
    }
  });
  return await response.json();
};

// GET /api/users/:id - Get user by ID
const getUserById = async (userId, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/api/users/${userId}`,
    {
      headers: {
        'Authorization': `Bearer ${token}`,
      }
    }
  );
  return await response.json();
};

// PUT /api/users/:id - Update user
const updateUser = async (userId, updateData, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/api/users/${userId}`,
    {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify(updateData)
    }
  );
  return await response.json();
};

// DELETE /api/users/:id - Delete user (Admin only)
const deleteUser = async (userId, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/api/users/${userId}`,
    {
      method: 'DELETE',
      headers: {
        'Authorization': `Bearer ${token}`,
      }
    }
  );
  return await response.json();
};
```

### Task Management Endpoints

```javascript
// POST /api/tasks - Create task
const createTask = async (taskData, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      title: taskData.title,
      description: taskData.description,
      assignedTo: taskData.userId,
      priority: taskData.priority, // 'low', 'medium', 'high'
      dueDate: taskData.dueDate,
      status: 'todo' // 'todo', 'inProgress', 'done'
    })
  });
  return await response.json();
};

// GET /api/tasks/user/:userId - Get user tasks
const getUserTasks = async (userId, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
    {
      headers: {
        'Authorization': `Bearer ${token}`,
      }
    }
  );
  return await response.json();
};

// PATCH /api/tasks/:id/status - Update task status
const updateTaskStatus = async (taskId, status, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
    {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({ status })
    }
  );
  return await response.json();
};

// POST /api/tasks/:id/time - Track time on task
const trackTaskTime = async (taskId, timeData, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({
        duration: timeData.duration, // in seconds
        date: timeData.date
      })
    }
  );
  return await response.json();
};
```

### Support Ticket Endpoints

```javascript
// POST /api/tickets - Create support ticket
const createTicket = async (ticketData, token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      subject: ticketData.subject,
      description: ticketData.description,
      category: ticketData.category, // 'technical', 'hr', 'admin'
      priority: ticketData.priority
    })
  });
  return await response.json();
};

// GET /api/tickets - Get all tickets (Admin)
const getAllTickets = async (token) => {
  const response = await fetch(`${process.env.REACT_APP_API_URL}/api/tickets`, {
    headers: {
      'Authorization': `Bearer ${token}`,
    }
  });
  return await response.json();
};

// PATCH /api/tickets/:id/assign - Assign ticket
const assignTicket = async (ticketId, assigneeId, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_API_URL}/api/tickets/${ticketId}/assign`,
    {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({ assignedTo: assigneeId })
    }
  );
  return await response.json();
};
```

## ML Service API

### Risk Detection

```javascript
// POST /api/ml/risk-detection
const detectUserRisk = async (userData, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-detection`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({
        userId: userData.userId,
        loginAttempts: userData.loginAttempts,
        failedLogins: userData.failedLogins,
        lastLogin: userData.lastLogin,
        activityPattern: userData.activityPattern
      })
    }
  );
  return await response.json();
  // Returns: { riskScore: 0.75, riskLevel: 'high', factors: [...] }
};
```

### Anomaly Detection

```javascript
// POST /api/ml/anomaly-detection
const detectAnomalies = async (activityData, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_ML_API_URL}/api/ml/anomaly-detection`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({
        userId: activityData.userId,
        loginTimes: activityData.loginTimes,
        accessPatterns: activityData.accessPatterns,
        dataVolume: activityData.dataVolume
      })
    }
  );
  return await response.json();
  // Returns: { isAnomaly: true, confidence: 0.89, anomalyType: 'unusual_access' }
};
```

### Burnout Detection

```javascript
// POST /api/ml/burnout-analysis
const analyzeBurnout = async (userId, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-analysis`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({
        userId: userId,
        workHours: { /* weekly hours data */ },
        taskLoad: { /* task counts and priorities */ },
        completionRates: { /* task completion data */ }
      })
    }
  );
  return await response.json();
  // Returns: { burnoutScore: 0.68, recommendation: 'reduce_workload', indicators: [...] }
};
```

### Ticket Classification

```javascript
// POST /api/ml/classify-ticket
const classifyTicket = async (ticketContent, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({
        subject: ticketContent.subject,
        description: ticketContent.description
      })
    }
  );
  return await response.json();
  // Returns: { category: 'technical', priority: 'high', suggestedAssignee: 'user_id' }
};
```

### Predictive Project Insights

```javascript
// POST /api/ml/project-insights
const getProjectInsights = async (projectData, token) => {
  const response = await fetch(
    `${process.env.REACT_APP_ML_API_URL}/api/ml/project-insights`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      body: JSON.stringify({
        projectId: projectData.projectId,
        tasks: projectData.tasks,
        teamMembers: projectData.teamMembers,
        deadline: projectData.deadline
      })
    }
  );
  return await response.json();
  // Returns: { delayProbability: 0.34, estimatedCompletion: '2026-05-15', recommendations: [...] }
};
```

## React Component Patterns

### User Dashboard Component

```javascript
import React, { useState, useEffect } from 'react';

const UserDashboard = () => {
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchUserTasks();
  }, []);

  const fetchUserTasks = async () => {
    try {
      const userId = getUserIdFromToken(token);
      const response = await fetch(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
        {
          headers: { 'Authorization': `Bearer ${token}` }
        }
      );
      const data = await response.json();
      setTasks(data.tasks);
    } catch (error) {
      console.error('Error fetching tasks:', error);
    } finally {
      setLoading(false);
    }
  };

  const moveTask = async (taskId, newStatus) => {
    try {
      await fetch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/status`,
        {
          method: 'PATCH',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({ status: newStatus })
        }
      );
      fetchUserTasks(); // Refresh
    } catch (error) {
      console.error('Error updating task:', error);
    }
  };

  return (
    <div className="dashboard">
      <h1>My Tasks</h1>
      <div className="kanban-board">
        <TaskColumn 
          title="To Do" 
          tasks={tasks.filter(t => t.status === 'todo')}
          onMove={moveTask}
        />
        <TaskColumn 
          title="In Progress" 
          tasks={tasks.filter(t => t.status === 'inProgress')}
          onMove={moveTask}
        />
        <TaskColumn 
          title="Done" 
          tasks={tasks.filter(t => t.status === 'done')}
          onMove={moveTask}
        />
      </div>
    </div>
  );
};
```

### Admin Analytics Dashboard

```javascript
import React, { useState, useEffect } from 'react';

const AdminDashboard = () => {
  const [analytics, setAnalytics] = useState(null);
  const [alerts, setAlerts] = useState([]);
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetchAnalytics();
    fetchAlerts();
  }, []);

  const fetchAnalytics = async () => {
    try {
      const response = await fetch(
        `${process.env.REACT_APP_API_URL}/api/admin/analytics`,
        {
          headers: { 'Authorization': `Bearer ${token}` }
        }
      );
      const data = await response.json();
      setAnalytics(data);
    } catch (error) {
      console.error('Error fetching analytics:', error);
    }
  };

  const fetchAlerts = async () => {
    try {
      const response = await fetch(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/alerts`,
        {
          headers: { 'Authorization': `Bearer ${token}` }
        }
      );
      const data = await response.json();
      setAlerts(data.alerts);
    } catch (error) {
      console.error('Error fetching alerts:', error);
    }
  };

  return (
    <div className="admin-dashboard">
      <h1>Organization Analytics</h1>
      
      {alerts.length > 0 && (
        <div className="alerts-section">
          <h2>AI Alerts</h2>
          {alerts.map(alert => (
            <div key={alert.id} className={`alert alert-${alert.severity}`}>
              <strong>{alert.type}:</strong> {alert.message}
            </div>
          ))}
        </div>
      )}

      {analytics && (
        <div className="analytics-grid">
          <MetricCard title="Total Users" value={analytics.totalUsers} />
          <MetricCard title="Active Tasks" value={analytics.activeTasks} />
          <MetricCard title="Open Tickets" value={analytics.openTickets} />
          <MetricCard 
            title="High Risk Users" 
            value={analytics.highRiskUsers}
            severity="warning"
          />
        </div>
      )}
    </div>
  );
};
```

### Time Tracking Component

```javascript
import React, { useState, useEffect } from 'react';

const TimeTracker = ({ taskId }) => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsedTime, setElapsedTime] = useState(0);
  const token = localStorage.getItem('token');

  useEffect(() => {
    let interval;
    if (isRunning) {
      interval = setInterval(() => {
        setElapsedTime(prev => prev + 1);
      }, 1000);
    }
    return () => clearInterval(interval);
  }, [isRunning]);

  const handleStart = () => {
    setIsRunning(true);
  };

  const handleStop = async () => {
    setIsRunning(false);
    
    // Save time to backend
    try {
      await fetch(
        `${process.env.REACT_APP_API_URL}/api/tasks/${taskId}/time`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({
            duration: elapsedTime,
            date: new Date().toISOString()
          })
        }
      );
      setElapsedTime(0);
    } catch (error) {
      console.error('Error saving time:', error);
    }
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
          <button onClick={handleStart}>Start</button>
        ) : (
          <button onClick={handleStop}>Stop</button>
        )}
      </div>
    </div>
  );
};
```

## Authentication Helper

```javascript
// utils/auth.js
import jwt_decode from 'jwt-decode';

export const getToken = () => {
  return localStorage.getItem('token');
};

export const setToken = (token) => {
  localStorage.setItem('token', token);
};

export const removeToken = () => {
  localStorage.removeItem('token');
};

export const getUserIdFromToken = (token) => {
  try {
    const decoded = jwt_decode(token);
    return decoded.userId;
  } catch (error) {
    console.error('Error decoding token:', error);
    return null;
  }
};

export const isTokenExpired = (token) => {
  try {
    const decoded = jwt_decode(token);
    return decoded.exp < Date.now() / 1000;
  } catch (error) {
    return true;
  }
};

export const getUserRole = (token) => {
  try {
    const decoded = jwt_decode(token);
    return decoded.role;
  } catch (error) {
    return null;
  }
};

// Protected route wrapper
export const withAuth = (Component) => {
  return (props) => {
    const token = getToken();
    
    if (!token || isTokenExpired(token)) {
      window.location.href = '/login';
      return null;
    }
    
    return <Component {...props} />;
  };
};

// Admin only wrapper
export const withAdminAuth = (Component) => {
  return (props) => {
    const token = getToken();
    
    if (!token || isTokenExpired(token)) {
      window.location.href = '/login';
      return null;
    }
    
    const role = getUserRole(token);
    if (role !== 'admin') {
      window.location.href = '/unauthorized';
      return null;
    }
    
    return <Component {...props} />;
  };
};
```

## Common Workflow Patterns

### Complete User Onboarding Flow

```javascript
// Register new user, assign initial task, analyze risk
const onboardNewUser = async (userData) => {
  const token = localStorage.getItem('token');
  
  try {
    // 1. Register user
    const registerResponse = await fetch(
      `${process.env.REACT_APP_API_URL}/api/auth/register`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(userData)
      }
    );
    const newUser = await registerResponse.json();
    
    // 2. Create onboarding task
    await fetch(`${process.env.REACT_APP_API_URL}/api/tasks`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      },
      body: JSON.stringify({
        title: 'Complete Onboarding',
        description: 'Review company policies and setup profile',
        assignedTo: newUser.userId,
        priority: 'high',
        dueDate: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
      })
    });
    
    // 3. Initial risk assessment
    const riskResponse = await fetch(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-detection`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          userId: newUser.userId,
          loginAttempts: 0,
          failedLogins: 0,
          lastLogin: new Date().toISOString(),
          activityPattern: 'new'
        })
      }
    );
    
    return {
      user: newUser,
      riskAssessment: await riskResponse.json()
    };
  } catch (error) {
    console.error('Onboarding error:', error);
    throw error;
  }
};
```

### Smart Ticket Creation with AI Classification

```javascript
const createSmartTicket = async (ticketData) => {
  const token = localStorage.getItem('token');
  
  try {
    // 1. Classify ticket using AI
    const classificationResponse = await fetch(
      `${process.env.REACT_APP_ML_API_URL}/api/ml/classify-ticket`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          subject: ticketData.subject,
          description: ticketData.description
        })
      }
    );
    const classification = await classificationResponse.json();
    
    // 2. Create ticket with AI suggestions
    const ticketResponse = await fetch(
      `${process.env.REACT_APP_API_URL}/api/tickets`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
          subject: ticketData.subject,
          description: ticketData.description,
          category: classification.category,
          priority: classification.priority,
          suggestedAssignee: classification.suggestedAssignee
        })
      }
    );
    const ticket = await ticketResponse.json();
    
    // 3. Auto-assign if high confidence
    if (classification.confidence > 0.8 && classification.suggestedAssignee) {
      await fetch(
        `${process.env.REACT_APP_API_URL}/api/tickets/${ticket.ticketId}/assign`,
        {
          method: 'PATCH',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({
            assignedTo: classification.suggestedAssignee
          })
        }
      );
    }
    
    return ticket;
  } catch (error) {
    console.error('Error creating smart ticket:', error);
    throw error;
  }
};
```

### Comprehensive User Analytics

```javascript
const getUserAnalytics = async (userId) => {
  const token = localStorage.getItem('token');
  
  try {
    // Fetch multiple analytics in parallel
    const [tasksRes, riskRes, burnoutRes] = await Promise.all([
      fetch(
        `${process.env.REACT_APP_API_URL}/api/tasks/user/${userId}`,
        { headers: { 'Authorization': `Bearer ${token}` } }
      ),
      fetch(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/risk-detection`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({ userId })
        }
      ),
      fetch(
        `${process.env.REACT_APP_ML_API_URL}/api/ml/burnout-analysis`,
        {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
          },
          body: JSON.stringify({ userId })
        }
      )
    ]);
    
    const tasks = await tasksRes.json();
    const risk = await riskRes.json();
    const burnout = await burnoutRes.json();
    
    return {
      taskStats: {
        total: tasks.length,
        completed: tasks.filter(t => t.status === 'done').length,
        inProgress: tasks.filter(t => t.status === 'inProgress').length,
        overdue: tasks.filter(t => new Date(t.dueDate) < new Date()).length
      },
      riskProfile: {
        score: risk.riskScore,
        level: risk.riskLevel,
        factors: risk.factors
      },
      burnoutIndicators: {
        score: burnout.burnoutScore,
        recommendation: burnout.recommendation,
        indicators: burnout.indicators
      }
    };
  } catch (error) {
    console.error('Error fetching user analytics:', error);
    throw error;
  }
};
```

## Configuration

### Backend Configuration (backend/config/config.js)

```javascript
module.exports = {
  port: process.env.PORT || 5000,
  mongoUri: process.env.MONGODB_URI,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpire: process.env.JWT_EXPIRE || '7d',
  mlServiceUrl: process.env.ML_SERVICE_URL || 'http://localhost:8000',
  corsOrigins: process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000'],
  maxLoginAttempts: 5,
  lockoutDuration: 15 * 60 * 1000, // 15 minutes
  auditLogEnabled: true
};
```

### Frontend Configuration (frontend/src/config/api.js)

```javascript
export const API_CONFIG = {
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:5000',
  mlServiceURL: process.env.REACT_APP_ML_API_URL || 'http://localhost:8000',
  timeout: 30000,
  retryAttempts: 3
};

export const getAuthHeaders = () => {
  const token = localStorage.getItem('token');
  return {
    'Content-Type': 'application/json',
    'Authorization': token ? `Bearer ${token}` : ''
  };
};
```

## Troubleshooting

### JWT Token Issues

```javascript
// Check if token is valid before making requests
const makeAuthenticatedRequest = async (url, options = {}) => {
  const token = getToken();
  
  if (!token) {
    throw new Error('No authentication token found');
  }
  
  if (isTokenExpired(token)) {
    removeToken();
    window.location.href = '/login';
    throw new Error('Token expired');
  }
  
  const headers = {
    ...options.headers,
    'Authorization': `Bearer ${token}`
  };
  
  const response = await fetch(url, { ...options, headers });
  
  if (response.status === 401) {
    removeToken();
    window.location.href = '/login';
    throw new Error('Authentication failed');
  }
  
  return response;
};
```

### ML Service Connection Issues

```javascript
// Fallback when ML service is unavailable
const getMLPrediction = async (endpoint, data, token) => {
  try {
    const response = await fetch(
      `${process.env.REACT_APP_ML_API_URL}${endpoint}`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization':
