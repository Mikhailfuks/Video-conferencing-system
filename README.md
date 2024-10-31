using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Sockets;
using System.Threading;
using System.Threading.Tasks;
using System.Windows.Forms;

namespace VideoConferenceSystem
{
    public class VideoConferenceForm : Form
    {
        private const int Port = 8888;

        private List<string> _connectedClients = new List<string>();
        private Dictionary<string, VideoClient> _clients = new Dictionary<string, VideoClient>();

        private TextBox _messageTextBox;
        private Button _sendButton;
        private ListBox _clientListBox;
        private Panel _videoPanel;

        public VideoConferenceForm()
        {
            InitializeComponent();
        }

        private void InitializeComponent()
        {
            // ... (Initialize UI elements here) ...
        }

        private async Task StartServer()
        {
            try
            {
                // Create a TCP listener
                var listener = new TcpListener(IPAddress.Any, Port);
                listener.Start();

                // Listen for incoming connections
                while (true)
                {
                    var client = await listener.AcceptTcpClientAsync();

                    // Handle the client connection
                    HandleClient(client);
                }
            }
            catch (Exception ex)
            {
                MessageBox.Show($"Error starting server: {ex.Message}");
            }
        }

        private async Task HandleClient(TcpClient client)
        {
            try
            {
                // Get the client's IP address and port
                var clientEndpoint = (IPEndPoint)client.Client.RemoteEndPoint;
                var clientAddress = $"{clientEndpoint.Address}:{clientEndpoint.Port}";

                // Add the client to the list of connected clients
                _connectedClients.Add(clientAddress);
                _clientListBox.Invoke((MethodInvoker)delegate { _clientListBox.Items.Add(clientAddress); });

                // Create a new VideoClient object to handle the client's video and chat
                var videoClient = new VideoClient(client, clientAddress);
                _clients.Add(clientAddress, videoClient);

                // Start the client's video and chat
                await videoClient.Start();

                // Listen for messages from the client
                while (true)
                {
                    var message = await videoClient.ReceiveMessage();
                    if (message != null)
                    {
                        // Display the message in the chat window
                        _messageTextBox.Invoke((MethodInvoker)delegate { _messageTextBox.AppendText($"{clientAddress}: {message}\r\n"); });
                    }
                }
            }
            catch (Exception ex)
            {
                // Handle the error
                RemoveClient(client);
                MessageBox.Show($"Client disconnected: {ex.Message}");
            }
        }

        private void RemoveClient(TcpClient client)
        {
            // Get the client's IP address and port
            var clientEndpoint = (IPEndPoint)client.Client.RemoteEndPoint;
            var clientAddress = $"{clientEndpoint.Address}:{clientEndpoint.Port}";

            // Remove the client from the list of connected clients
            _connectedClients.Remove(clientAddress);
            _clientListBox.Invoke((MethodInvoker)delegate { _clientListBox.Items.Remove(clientAddress); });

            // Remove the client from the dictionary
            if (_clients.ContainsKey(clientAddress))
            {
                _clients.Remove(clientAddress);
            }

            // Stop the client's video and chat
            if (_clients.TryGetValue(clientAddress, out var videoClient))
            {
                videoClient.Stop();
            }
        }

        private void SendButton_Click(object sender, EventArgs e)
        {
            // Get the message from the text box
            var message = _messageTextBox.Text;

            // Send the message to all connected clients
            foreach (var clientAddress in _connectedClients)
            {
                if (_clients.TryGetValue(clientAddress, out var videoClient))
                {
                    videoClient.SendMessage(message);
                }
            }

            // Clear the text box
            _messageTextBox.Clear();
        }

        private void ClientListBox_DoubleClick(object sender, EventArgs e)
        {
            // Get the selected client's IP address and port
            var selectedClient = _clientListBox.SelectedItem?.ToString();
            if (selectedClient != null)
            {
                // Display the client's video in the video panel
                if (_clients.TryGetValue(selectedClient, out var videoClient))
                {
                    _videoPanel.Controls.Clear();
                    _videoPanel.Controls.Add(videoClient.GetVideoControl());
                }
            }
        }

        // ... (Implement other UI event handlers and methods) ...
    }

    public class VideoClient
    {
        private readonly TcpClient _client;
        private readonly string _clientAddress;

        private NetworkStream _stream;
        private CancellationTokenSource _cts;

        private Task _receiveTask;
        private Task _videoTask;

        private WebCamControl _videoControl;

        public VideoClient(TcpClient client, string clientAddress)
        {
            _client = client;
            _clientAddress = clientAddress;

            // Initialize video control (replace with your desired video control)
            _videoControl = new WebCamControl();
        }

        public async Task Start()
        {
            // Get the network stream
            _stream = _client.GetStream();

            // Create a cancellation token source
            _cts = new CancellationTokenSource();

            // Start receiving messages
            _receiveTask = ReceiveMessagesAsync(_cts.Token);

            // Start the video stream
            _videoTask = StartVideoStreamAsync(_cts.Token);
        }

        public void Stop()
        {
            // Cancel the receive and video tasks
            _cts.Cancel();

            // Dispose of the video control
            _videoControl.Dispose();
        }

        public async Task SendMessage(string message)
        {
            // Encode the message to bytes
            var messageBytes = Encoding.UTF8.GetBytes(message);

            // Send the message over the network stream
            await _stream.WriteAsync(messageBytes, 0, messageBytes.Length);
        }

        public async Task<string> ReceiveMessage()
        {
            // Read the message length
            var lengthBytes = new byte[4];
            await _stream.ReadAsync(lengthBytes, 0, lengthBytes.Length);
            int messageLength = BitConverter.ToInt32(lengthBytes, 0);

            // Read the message
            var messageBytes = new byte[messageLength];
            await _stream.ReadAsync(messageBytes, 0, messageLength);

            // Decode the message from bytes
            return Encoding.UTF8.GetString(messageBytes);
        }

        private async Task ReceiveMessagesAsync(CancellationToken cancellationToken)
        {
            try
            {
                while (!cancellationToken.IsCancellationRequested)
                {
                    // Receive a message from the client
                    var message = await ReceiveMessage();
                    if (message != null)
                    {
                        // Handle the message
                        // ...
                    }
                }
            }
            catch (Exception ex)
            {
                // Handle the error
                // ...
            }
        }

        private async Task StartVideoStreamAsync(CancellationToken cancellationToken)
        {
            try
            {
                // Start the video capture (replace with your video capture logic)
                _videoControl.StartCapture();

                // Send video frames over the network stream
                while (!cancellationToken.IsCancellationRequested)
                {
                    // Capture a frame from the video source
                    var frame = _videoControl.GetFrame();

                    // Encode the frame to bytes
                    var frameBytes = // ... (Encode frame to bytes) ...

                    // Send the frame over the network stream
                    await _stream.WriteAsync(frameBytes, 0, frameBytes.Length);
                }
            }
            catch (Exception ex)
            {
                // Handle the error

            }
        }

        public Control GetVideoControl()
        {
            return _videoControl;
        }
    }

    // (Implement other classes like WebCamControl, etc.) 
}
