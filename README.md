# Real-Time Data Communication with Socket.IO

Socket.IO is a library that enables real-time, bidirectional, and event-based communication between web clients and servers. It is primarily used in applications that require real-time updates, such as chat applications, online gaming, collaborative tools, and dashboards. Socket.IO consists of two parts:

- **Server-side library for Node.js**
- **Client-side library that runs in the browser**

Both components have nearly identical APIs, allowing for consistent communication between server and client.

---

## Features

- **Real-time communication**: Enables instant messaging and real-time data transfer.
- **Cross-browser support**: Works on all major browsers, including those that do not support WebSocket by providing fallbacks.
- **Automatic reconnection**: Automatically attempts to reconnect when a connection is lost.
- **Broadcasting**: Supports sending messages to multiple clients.

---

## Installation

To use Socket.IO, you need to install it on both the server and client sides.

### Client-side Installation

You can install the client library using npm, or you can include it directly in your HTML.
- Using npm:

```bash
npm install socket.io-client
```

- Using CDN:

```html
<script src="https://cdn.socket.io/4.6.1/socket.io.min.js"></script>
```

### Server-side Installation

To install Socket.IO on the server side (Node.js environment), run:

```bash
npm install socket.io
```

### Setting Up the Server

First, create a basic server using Express and integrate Socket.IO:

```javascript
const express = require('express');
const http = require('http');
const { Server } = require('socket.io'); // Import Socket.IO

const app = express();
const server = http.createServer(app);
const io = new Server(server); // Initialize Socket.IO server

io.on('connection', (socket) => {
  console.log('a user connected');
  socket.on('disconnect', () => {
    console.log('user disconnected');
  });
});

server.listen(3000, () => {
  console.log('server listening on :3000');
});
```

### Setting Up the Client

On the client side, connect to the server using the Socket.IO client library:

```typescript
import { io, Socket } from 'socket.io-client'; // Import Socket.IO Client

export class DataService {
  private socket: Socket;
  
  constructor() {
    this.socket = io('http://localhost:5000'); // Connect to Socket.IO server

    this.socket.on('connect', () => {
      console.log('Connected to server');
    });

    this.socket.on('disconnect', () => {
      console.log('Disconnected from server');
    });
  }
}
```

### Emitting Events

Socket.IO allows you to send and receive custom events.

### Server-side:

```javascript
io.on('connection', (socket) => {
  socket.emit('welcome', 'Hello! Welcome to the server.'); // Emit event from server
});
```

### Client-side:

```typescript
this.socket.on('welcome', (message) => {
  console.log(message); // Output: Hello! Welcome to the server.
});
```

### Sample Code

### Server-side Code

```javascript
const express = require('express');
const cors = require('cors');
const http = require('http');
const { Server } = require('socket.io'); // Import Socket.IO Server
const { dbConnection } = require('./server');
const DeviceData = require('./models/wat_deviceData');

const app = express();
const port = 5000;
const allowedOrigins = ['http://localhost:4200', 'http://localhost:59998/'];

app.use(express.json());

// Create HTTP server
const server = http.createServer(app);

// Create Socket.IO server with CORS options
const io = new Server(server, {
  cors: {
    origin: allowedOrigins,
    methods: ['GET', 'POST'],
    credentials: true,
  },
});

dbConnection();

io.on('connection', (socket) => { // Handle client connection
  console.log(socket.id);
  socket.clientname = 'Appland';

  const sendAggregatedData = () => {
    DeviceData.aggregate([
      { $match: { clientName: socket.clientname } },
      { $group: { _id: '$DeviceName', count: { $sum: 1 } } },
      { $project: { DeviceName: '$_id', count: 1, _id: 0 } },
      { $sort: { count: -1 } },
    ])
      .then((data) => socket.emit('DeviceData', data)) // Emit data to client
      .catch((error) => console.error('Error sending aggregated data:', error));
  };

  sendAggregatedData();

  socket.on('clientName', (data) => {
    socket.clientname = data;
    sendAggregatedData();
  });

  const watchCollections = (watcher, handler) => {
    watcher
      .on('change', (change) => {
        handler(change);
      })
      .on('error', (error) => console.error('Error watching collection:', error));
  };

  watchCollections(DeviceData.watch(), () => sendAggregatedData());

  socket.on('disconnect', () => {
    console.log('Disconnected from server');
  });
});

server.listen(port, () => {
  console.log(`Server is running on port ${port}`);
});
```

### Client-side Code
Service.ts

```typescript
import { io, Socket } from 'socket.io-client'; // Import Socket.IO client

export class DataService {
  private socket: Socket;

  constructor() {
    this.socket = io('http://localhost:5000'); // Connect to Socket.IO server
    this.socket.on('connect', () => {
      console.log('Socket connected:', this.socket.id);
    });
  }

  emitSelectedClient(clientName: string) {
    this.socket.emit('clientName', clientName); // Emit clientName to server
  }

  public onDataUpdate(callback: (data: any) => void): void {
    this.socket.on('DeviceData', callback); // Listen for DeviceData event from server
  }
}
```

component.ts

```typescript
import { DataService } from '../../../shared/services/dashboard.service';

export class DashboardOverviewComponent implements OnDestroy, OnInit {
  Devicedata: any[] = [];

  constructor(public dataService: DataService) {}

  ngOnInit(): void {
    this.dataService.onDataUpdate((data) => {
      this.Devicedata = data;
      this.getDeviceData();
    });
  }

  getDeviceData() {
    const deviceData = this.Devicedata
      .filter((entry) => entry.DeviceName)
      .map((entry) => ({
        name: entry.DeviceName,
        y: entry.count,
      }));

    this.totalDeviceCount = deviceData.reduce(
      (total, entry) => total + entry.y,
      0
    );

    const pieChartOptions: Highcharts.Options = {
      credits: { enabled: false },
      chart: { type: 'pie', backgroundColor: 'transparent' },
      title: { text: '' },
      tooltip: { pointFormat: '{series.name}: <b>{point.y}</b>' },
      plotOptions: {
        pie: {
          innerSize: '80%',
          borderWidth: 0,
          depth: 10,
          animation: false,
          dataLabels: {
            enabled: true,
            color: '#000000',
            style: { textOutline: 'none' },
          },
        },
      },
      colors: ['#ffc107', '#052288', '#BBC1D2'],
      series: [{ type: 'pie', name: 'Count', data: deviceData }],
    };

    Highcharts.chart('pieChartContainer', pieChartOptions);

    const totalCountElement = document.getElementById('total-count');
    if (totalCountElement) {
      totalCountElement.innerText = 'Total Device Count: ' + this.totalDeviceCount;
    }
  }

  onDashboardChange(selectedClient: string) {
    this.dataService.emitSelectedClient(selectedClient); // Emit client change to server
  }
}
```