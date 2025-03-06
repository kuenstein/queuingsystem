const express = require('express');
const cors = require('cors');
const WebSocket = require('ws');
const { Parser } = require('json2csv');
const path = require('path');
const fs = require('fs');

const app = express();
app.use(cors());
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

let queues = {
    Charging: [],
    Releasing: [],
    Extraction: []
};

let currentServing = {
    Charging: null,
    Releasing: null,
    Extraction: null
};

let lastServed = {
    Charging: null,
    Releasing: null,
    Extraction: null
};

let queueNumber = 0;
let servedHistory = [];
let totalServed = 0;
let totalWaitTime = 0;
let currentAnnouncement = "";

let settings = {
    averageServiceTime: 5,
    maxQueueLength: 100
};

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
    console.log('Client connected');
});

function notifyClients(message) {
    wss.clients.forEach((client) => {
        if (client.readyState === WebSocket.OPEN) {
            client.send(message);
        }
    });
}

function generateQueueNumber() {
    return ++queueNumber;
}

function addToQueue(service, user) {
    if (queues[service]) {
        if (queues[service].length < settings.maxQueueLength) {
            queues[service].push(user);
        } else {
            console.error(`Queue for ${service} is full.`);
        }
    }
}

function callNextNumber(service) {
    let nextUser = queues[service].shift();
    if (nextUser) {
        currentServing[service] = nextUser;
        notifyClients(`Now serving number ${nextUser.number} at ${service}`);
        return nextUser;
    }
    return null;
}

app.get('/queue', (req, res) => {
    const status = {};
    for (const service in queues) {
        const waitingUsers = queues[service];
        const estimatedWaitTime = (waitingUsers.length * settings.averageServiceTime);
        status[service] = {
            current: currentServing[service] ? { number: currentServing[service].number } : null,
            waiting: waitingUsers.map(user => user.number),
            estimatedWaitTime: estimatedWaitTime
        };
    }
    res.json(status);
});

app.post('/queue', (req, res) => {
    const { service } = req.body;
    const user = { number: generateQueueNumber(), service };
    addToQueue(service, user);
    res.json({ number: user.number });
});

app.post('/call', (req, res) => {
    const { service } = req.body;
    const currentUser = callNextNumber(service);
    if (currentUser) {
        servedHistory.push(currentUser);
        totalServed++;
        totalWaitTime += settings.averageServiceTime;
        lastServed[service] = currentUser; // Store the last served number
        res.json({ current: currentUser.number });
    } else {
        res.json({ current: null });
    }
});

app.post('/recall', (req, res) => {
    const { service } = req.body;
    if (lastServed[service]) {
        res.json({ lastNumber: lastServed[service].number });
    } else {
        res.json({ lastNumber: null });
    }
});

app.post('/announcement', (req, res) => {
    const { announcement } = req.body;
    currentAnnouncement = announcement;
    notifyClients(`New Announcement: ${announcement}`);
    res.status(200).send();
});

app.get('/announcement', (req, res) => {
    res.json({ announcement: currentAnnouncement });
});

// Function to export queue data to CSV
function exportQueueData() {
    const csvData = [];
    for (const service in queues) {
        queues[service].forEach(user => {
            csvData.push({ service, number: user.number });
        });
    }

    if (csvData.length === 0) {
        console.log('No queue data available to export.');
        return;
    }

    const parser = new Parser();
    const csv = parser.parse(csvData);
    const filePath = path.join(__dirname, 'queue_data.csv');

    fs.writeFileSync(filePath, csv);
    console.log('Queue data exported successfully on server restart.');
}

// API endpoint for manual export
app.get('/export', (req, res) => {
    const csvData = [];
    for (const service in queues) {
        queues[service].forEach(user => {
            csvData.push({ service, number: user.number });
        });
    }

    if (csvData.length === 0) {
        return res.status(400).send('No data available to export.');
    }

    const parser = new Parser();
    const csv = parser.parse(csvData);

    res.header('Content-Type', 'text/csv');
    res.attachment('queue_data.csv');
    res.send(csv);
});

app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).send('Something broke!');
});

// Start the server and automatically export data on restart
app.listen(3000, () => {
    console.log('Server is running on port 3000');
    exportQueueData(); // Export queue data on server start
});



app.post('/reset-queue', (req, res) => {
    try {
        // Reset the queue data (modify based on your storage method)
        queueData = {}; // If using an object in memory

        // If storing queue data in a JSON file:
        fs.writeFileSync('queueData.json', JSON.stringify({}), 'utf8');

        console.log("Queue has been reset.");
        res.status(200).json({ message: "Queue reset successfully" });
    } catch (error) {
        console.error("Error resetting queue:", error);
        res.status(500).json({ error: "Failed to reset queue" });
    }
});


