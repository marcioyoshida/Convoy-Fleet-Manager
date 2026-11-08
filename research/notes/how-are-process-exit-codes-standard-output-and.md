# How are process exit codes, standard output, and

> Source note (DOCX). AI-generated take collected during the Convoy design debate; converted to text verbatim. The critical review lives in `docs/debate-review.md`.

In a traditional POSIX environment, standard output (stdout), standard error (stderr), and process exit codes are handled directly by the kernel via file descriptors (FD 1 and 2) and the waitpid() system call, routed straight to the local terminal device (/dev/tty).

In a Kubernetes-backed shell, because processes are isolated inside remote containers distributed across cluster nodes, capturing and streaming these signals requires a multi-tier log aggregation and API streaming pipeline.

## 1. Capturing stdout and stderr at the Container Layer

When a process writes to stdout or stderr, it does not hit a physical monitor or terminal window. Instead:

Container Runtime Interception: The container runtime (containerd or CRI-O) intercepts the container's standard streams at the file descriptor level.

Node-Level Logging: The runtime redirects these streams into standardized JSON log files stored on the worker node's local disk (typically under /var/log/pods/).

Separation of Streams: Unlike basic log collectors that merge streams, the container runtime preserves the separation of stdout and stderr by prefixing or framing the log payloads with stream identifiers, ensuring error outputs can still be routed correctly back to the user's error channel.

## 2. Streaming Across the Network (The API Transport Layer)

To stream logs in real-time back to the user's shell terminal (for commands like tail -f or active pipelines), the shell client communicates with the Kubernetes API Server:

HTTP Chunked Transfer / WebSockets: When a user executes a command, the shell opens a persistent streaming connection to the Kubernetes API server's log endpoint (/api/v1/namespaces/{ns}/pods/{pod}/log?follow=true).

The Kubelet Proxy: The API server proxies this request down to the kubelet running on the specific worker node hosting the pod.

Client-Side Demultiplexing: The shell client receives the multiplexed stream, unpacks it, and writes the output directly to the user's local terminal window, making remote container execution feel identical to running a local background process.

## 3. Capturing and Returning Process Exit Codes

In a shell, knowing when a process finishes and how it exited ($? in Bash) is critical for scripting and pipeline logic (e.g., cmd1 && cmd2).

Container Lifecycle Completion: When the main process inside a container exits, the container status transitions to Terminated.

Exit Code Storage: The container runtime captures the exact integer exit status (e.g., 0 for success, 1 or higher for errors, or signal offsets like 137 for OOMKilled/SIGKILL) and writes it into the pod's API status object (containerStatuses[0].state.terminated.exitCode).

Shell Translation: The shell client continuously watches the pod status. Once the pod reaches a terminal phase (Succeeded or Failed), the shell extracts the integer exit code, populates the session's local exit status variable, and unblocks the prompt.

## 4. Interactive Sessions (stdin and TTY Allocation)

For interactive commands (like running an interactive Python REPL or vim), streaming output in one direction is not enough; the shell must also stream user keystrokes (stdin) back into the remote container.

SPDY / WebSockets Multiplexing: Kubernetes supports interactive execution via kubectl exec, which upgrades an HTTP connection to a multiplexed protocol (historically SPDY, moving toward WebSockets/HTTP2).

Pseudo-Terminal (pty) Allocation: The shell request allocates a pseudo-terminal (-it flags) on the container runtime, mapping terminal dimensions (rows and columns) so that line wrapping and interactive text editors render perfectly across the distributed network link.
