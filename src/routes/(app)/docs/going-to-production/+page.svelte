<script>
    import HeadingLink from "@/components/HeadingLink.svelte";
    import CommonHelper from "@/utils/CommonHelper";
    import CodeBlock from "@/components/CodeBlock.svelte";
    import Toc from "@/components/Toc.svelte";
</script>

<Toc />

<HeadingLink title="Deployment strategies" />

<HeadingLink title="Minimal setup" tag="h5" />
<p>
    One of the best PocketBase features is that it's completely portable. This mean that it doesn't require
    any external dependency and
    <strong>could be deployed by just uploading the executable on your server</strong>.
</p>
<p>
    Here is an example for starting a production HTTPS server (auto managed TLS with Let's Encrypt) on clean
    Ubuntu 22.04 installation.
</p>
<ol>
    <li value="0">
        <p>Consider the following app directory structure:</p>
        <CodeBlock
            language="html"
            content={`
                myapp/
                    pb_migrations/
                    pb_hooks/
                    pocketbase
            `}
        />
    </li>
    <li>
        <p>
            Upload the binary and anything else related to your remote server, for example using
            <strong>rsync</strong>:
        </p>
        <CodeBlock
            content={`
                rsync -avz -e ssh /local/path/to/myapp root@YOUR_SERVER_IP:/root/pb
            `}
        />
    </li>
    <li>
        <p>Start a SSH session with your server:</p>
        <CodeBlock
            content={`
                ssh root@YOUR_SERVER_IP
            `}
        />
    </li>
    <li>
        <p>Start the executable (specifying a domain name will issue a Let's encrypt certificate for it)</p>
        <CodeBlock
            content={`
              [root@dev ~]$ /root/pb/pocketbase serve yourdomain.com
            `}
        />
        <blockquote>
            <p>
                Notice that in the above example we are logged in as <strong>root</strong> which allow us to
                bind to the
                <strong>privileged 80 and 443 ports</strong>.
                <br />
                For <strong>non-root</strong> users usually you'll need special privileges to be able to do
                that. You have several options depending on your OS - <code>authbind</code>,
                <code>setcap</code>,
                <code>iptables</code>, <code>sysctl</code>, etc. Here is an example using <code>setcap</code>:
            </p>
            <CodeBlock
                content={`
                    [myuser@dev ~]$ sudo setcap 'cap_net_bind_service=+ep' /root/pb/pocketbase
                `}
            />
        </blockquote>
    </li>
    <li>
        <p>(Optional) systemd service</p>
        <p>
            You can skip step 3 and create a <strong>Systemd service</strong>
            to allow your application to start/restart on its own.
            <br />
            Here is an example service file (usually created in
            <code>/lib/systemd/system/pocketbase.service</code>):
        </p>
        <CodeBlock
            content={`
                [Unit]
                Description = pocketbase

                [Service]
                Type           = simple
                User           = root
                Group          = root
                LimitNOFILE    = 4096
                Restart        = always
                RestartSec     = 5s
                StandardOutput = append:/root/pb/errors.log
                StandardError  = append:/root/pb/errors.log
                ExecStart      = /root/pb/pocketbase serve yourdomain.com

                [Install]
                WantedBy = multi-user.target
            `}
        />
        <p>
            After that we just have to enable it and start the service using <code>systemctl</code>:
        </p>
        <CodeBlock
            content={`
                [root@dev ~]$ systemctl enable pocketbase.service
                [root@dev ~]$ systemctl start pocketbase
            `}
        />
    </li>
</ol>

<HeadingLink title="Using reverse proxy" tag="h5" />
<p>
    If you plan hosting multiple applications on a single server or need finer network controls (rate limiter,
    IPs whitelisting, etc.), you could always put PocketBase behind a reverse proxy such as
    <em>NGINX</em> or <em>Apache</em>. Here is a minimal <em>NGINX</em> sample configuration:
</p>
<CodeBlock
    language="html"
    content={`
    server {
        listen 80;
        server_name example.com;
        client_max_body_size 10M;

        location / {
            # check http://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive
            proxy_set_header Connection '';
            proxy_http_version 1.1;
            proxy_read_timeout 360s;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # enable if you are serving under a subpath location
            # rewrite /yourSubpath/(.*) /$1  break;

            proxy_pass http://127.0.0.1:8090;
        }
    }
    `}
/>

<HeadingLink title="Using Docker" tag="h5" />
<p>
    Some hosts (eg. <a href="https://fly.io" target="_blank" rel="noopener noreferrer">fly.io</a>) use Docker
    for deployments. PocketBase doesn't have an official Docker image yet, but you could use the below
    Dockerfile as an example:
</p>
<CodeBlock
    language="html"
    content={`
        FROM alpine:latest

        ARG PB_VERSION=` +
        (import.meta.env.PB_VERSION.startsWith("v")
            ? import.meta.env.PB_VERSION.substring(1)
            : import.meta.env.PB_VERSION) +
        `

        RUN apk add --no-cache \\
            unzip \\
            ca-certificates

        # download and unzip PocketBase
        ADD https://github.com/pocketbase/pocketbase/releases/download/v\${PB_VERSION}/pocketbase_\${PB_VERSION}_linux_amd64.zip /tmp/pb.zip
        RUN unzip /tmp/pb.zip -d /pb/

        # uncomment to copy the local pb_migrations dir into the image
        # COPY ./pb_migrations /pb/pb_migrations

        # uncomment to copy the local pb_hooks dir into the image
        # COPY ./pb_hooks /pb/pb_hooks

        EXPOSE 8080

        # start PocketBase
        CMD ["/pb/pocketbase", "serve", "--http=0.0.0.0:8080"]
    `}
/>
<p>
    To persist your data you need to mount a volume at <code>/pb/pb_data</code>.
</p>
<p class="txt-hint">
    <em>
        For a full example you could check the
        <a
            href="https://github.com/pocketbase/pocketbase/discussions/537"
            target="_blank"
            rel="noopener noreferrer"
        >
            "Host for free on Fly.io"
        </a>
        or
        <a
            href="https://github.com/pocketbase/pocketbase/discussions/2856"
            target="_blank"
            rel="noopener noreferrer"
        >
            "Host for free on Hop.io"
        </a>
        guides.
    </em>
</p>

<HeadingLink title="Backup and Restore" />
<p />
<p>
    PocketBase v0.16+ comes with built-in backups and restore APIs that could be accessed from the Admin UI (<em
        >Settings</em
    >
    > <em>Backups</em>):
</p>
<img src="/images/screenshots/backups.png" alt="Backups settings screenshot" class="screenshot m-b-xs" />
<p>
    <strong>
        Note that the application will be temporary set in read-only mode during the backup's ZIP generation.
    </strong>
    <br />
    Backups can be stored locally (default) or in an external S3 storage.
</p>
<p>
    Alternatively, you can always manually copy your <code>pb_data</code> directory
    <em>(for transactional safety make sure that the application is not running)</em>.
</p>

<HeadingLink title="Recommendations" />

<header class="highlighted-title bg-danger-alt m-t-0">
    <span class="label label-primary">highly recommended</span>
    <HeadingLink title="Use SMTP mail server" tag="h5" />
</header>
<p>
    By default, PocketBase uses the internal Unix <code>sendmail</code> command for sending emails.
    <br />
    While it's OK for development, it's not very useful for production, because your emails most likely will get
    marked as spam or even fail to deliver.
</p>
<p>
    To avoid deliverability issues, consider using a local SMTP server or an external mail service like
    <a href="https://www.mailersend.com/" target="_blank" rel="noreferrer noopener">MailerSend</a>,
    <a href="https://www.brevo.com/" target="_blank" rel="noreferrer noopener">Brevo</a>,
    <a href="https://sendgrid.com/" target="_blank" rel="noreferrer noopener">SendGrid</a>,
    <a href="https://www.mailgun.com/" target="_blank" rel="noreferrer noopener">Mailgun</a>,
    <a href="https://aws.amazon.com/ses/" target="_blank" rel="noreferrer noopener">AWS SES</a>, etc.
</p>
<p>
    Once you've decided on a mail service, you could configure the PocketBase SMTP settings from the Admin UI
    (<em>Settings</em> > <em>Mail settings</em>):
</p>
<img src="/images/screenshots/smtp-settings.png" alt="SMTP settings screenshot" class="screenshot m-b-xs" />

<header class="highlighted-title bg-warning-alt">
    <span class="label label-primary">optional</span>
    <HeadingLink title="Increase the open file descriptors limit" tag="h5" />
</header>
<p class="txt-hint txt-bold">
    The below instructions are for Linux but other operating systems have similar mechanism.
</p>
<p>
    Unix uses <em>"file descriptors"</em> also for network connections and most systems have a default limit
    of ~ 1024.
    <br />
    If your application has a lot of concurrent realtime connections, it is possible that at some point you would
    get an error such as: <code>Too many open files</code>.
</p>
<p>
    One way to mitigate this is to check your current account resource limits by running
    <code>ulimit -a</code> and find the parameter you want to change. For example, if you want to increase the
    open files limit (<em>-n</em>), you could run
    <code>ulimit -n 4096</code> before starting PocketBase.
</p>

<header class="highlighted-title bg-warning-alt">
    <span class="label label-primary">optional</span>
    <HeadingLink title="Enable settings encryption" tag="h5" />
</header>
<p class="txt-bold txt-hint">It is fine to ignore the below if you are not sure whether you need it.</p>
<p>
    By default, PocketBase stores the applications settings in the database as plain JSON text, including the
    secret keys for the OAuth2 clients and the SMTP password.
</p>
<p>
    While this is not a security issue on its own (PocketBase applications live entirely on a single server
    and its expected only authorized users to have access to your server and application data), in some
    situations it may be a good idea to store the settings encrypted in case someone get their hands on your
    database file (eg. from an external stored backup).
</p>
<p>To store your PocketBase settings encrypted:</p>
<ol>
    <li class="m-b-10">
        Create a new environment variable and <strong>set a random 32 characters</strong> string as its value.
        <br />
        <span class="txt-hint">
            eg. add
            <code>export PB_ENCRYPTION_KEY="{CommonHelper.randomString(32)}"</code>
            in your shell profile file
        </span>
    </li>
    <li>
        Start the application with <code>--encryptionEnv=YOUR_ENV_VAR</code> flag.
        <br />
        <span class="txt-hint">
            eg. <code>pocketbase serve --encryptionEnv=PB_ENCRYPTION_KEY</code>
        </span>
    </li>
</ol>
