% Your 5G Lab — A Plain-English Daily Guide
% How to switch it on, use it, test it, and switch it off — no experience needed

# Read this first (2 minutes)

This guide lets **anyone** run the 5G lab on their own, start to finish, without asking for help.
You do **not** need to understand networking. If you can type text and press **Enter**, you can do
everything here.

**What is this lab, in one sentence?** It's a tiny, private mobile-phone network running entirely on
one computer — a pretend phone connects to a pretend cell tower, which connects to "the network,"
and then the pretend phone can reach the internet, exactly like your real phone does over 4G/5G.

**The three characters you'll keep hearing about:**

- **The phone** — called the **UE**. (UE = "User Equipment," a fancy word for a phone.)
- **The cell tower** — called the **gNB**. (This is the 5G radio mast.)
- **The network / SIM system** — called the **Core**. This is the part that checks the SIM card,
  says "yes you're allowed," and connects the phone to the internet.

**How you'll work every day:** you switch the Core on, switch the tower on, switch the phone on, and
then test that the phone can reach the internet. At the end you switch it all off. That's it.

**One golden rule:** you will open a few **black windows** on screen (these are called *terminals* —
think of them as a place to type commands). **Each character gets its own window.** Don't put two in
the same window, or one of them shuts down. This guide tells you exactly which window is which.

\newpage

# What you need before you start

You need the computer that already has the lab installed on it — a **Ubuntu (Linux)** machine (it
may be a "virtual machine," which is just a Linux computer running inside your normal computer). If
someone set this up for you, that's all in place already.

You do **not** need to install anything. You do **not** need the internet to be fast. You just need
the machine turned on.

**Three things you'll use, explained simply:**

| You'll hear… | It just means… |
|--------------|----------------|
| **Terminal** | A black window where you type a command and press Enter. |
| **Command** | A line of text you type to make the computer do something. Type it exactly. |
| **`cd` / `up` / `ping`** | Ordinary commands. You don't need to know how they work — just type them. |

**How to type a command:** click inside the black window once, type the line exactly as shown
(spaces matter), then press **Enter**. To paste instead of type, use **Ctrl+Shift+V** inside a
terminal (not the usual Ctrl+V).

\newpage

# The daily routine — 6 simple steps

Do these in order every time you want to use the lab. Each step says **what to type**, **what you'll
see**, and **what it means**.

## Step 1 — Turn on the "network" (the Core)

**Open a terminal window** (look for the "Terminal" app; its icon is a small black square). Then
type these two lines, pressing Enter after each:

```bash
cd docker-compose
docker compose -f sa-deploy.yaml up -d
```

- The first line means *"go into the folder that holds the network."*
- The second line means *"switch the whole network on in the background."*

**What you'll see:** a list of names (amf, smf, upf, nrf, mongo, and others) each ending in
**Started** or **Running**. That's every part of the network waking up.

**What it means:** the "SIM system" is now on and waiting for a phone. Give it about **30 seconds**
to settle before the next step.

> If you see red text or "error," don't panic — jump to the **"If something goes wrong"** page at
> the end. Most of the time the fix is simple.

## Step 2 — Add the SIM card (only the first time)

Your pretend phone needs a SIM card registered in the network — just like a real phone. **You only
do this once.** If someone already did it, skip to Step 3.

1. Open the web browser on the machine.
2. In the address bar type: **`http://127.0.0.1:9999`** and press Enter.
3. Log in with username **`admin`** and password **`1423`**.
4. Click **Subscriber**, then the **+** (add) button.
5. In the **IMSI** box type: **`999700000000001`** (the SIM's ID number).
6. Leave the other boxes at their default values and click **Save**.

**What it means:** the network now "knows" your phone's SIM and will let it connect.

## Step 3 — Turn on the cell tower (the gNB)

**Open a NEW terminal window** (a second black window — this is important; the tower needs its own
window). Type:

```bash
cd ~/private-5g/UERANSIM
./build/nr-gnb -c config/open5gs-gnb.yaml
```

**What you'll see:** several lines of text, ending with something like **"NG Setup successful."**

**What it means:** the tower has connected to the network. **Leave this window open** — if you close
it, the tower switches off. Just move it aside.

## Step 4 — Turn on the phone (the UE)

**Open a THIRD terminal window.** Type (this one starts with `sudo`, which may ask for the machine's
password — type it and press Enter; the password won't show as you type, that's normal):

```bash
cd ~/private-5g/UERANSIM
sudo ./build/nr-ue -c config/open5gs-ue.yaml
```

**What you'll see:** lines ending with something like **"Connection setup for PDU session … is
successful"** and a mention of **`uesimtun0`**.

**What it means:** the phone is now connected and has been given an internet connection.
**Leave this window open too.**

## Step 5 — Test that the phone can reach the internet

**Open a FOURTH terminal window.** Type:

```bash
ping -I uesimtun0 -c 4 8.8.8.8
```

- This means *"using the phone's connection, send 4 test messages to the internet and see if they
  come back."* (`8.8.8.8` is a well-known internet address that always answers.)

**What you'll see (success):** four lines saying **"64 bytes from 8.8.8.8 …"** and at the end
**"0% packet loss."**

**What it means:** 🎉 **it works** — your pretend phone is on the internet through your own private
5G network. `0% packet loss` is the phrase that means "perfect, nothing was dropped."

**If you see "100% packet loss":** the data path didn't come up. Go to the troubleshooting page.

## Step 6 — Shut everything down at the end of the day

When you're finished, switch things off in the reverse order so nothing is left running.

1. Go to the **phone** window (Step 4) and press **Ctrl+C**. This stops the phone.
2. Go to the **tower** window (Step 3) and press **Ctrl+C**. This stops the tower.
3. Go back to the **first** window and type:

```bash
cd docker-compose
docker compose -f sa-deploy.yaml down
```

This switches the network off cleanly. You can now close the windows.

**What it means:** everything is off and your machine is back to normal. Nothing keeps running in the
background.

\newpage

# A checklist you can print

Tick each box as you go. This is the whole daily routine on one page.

**Starting up**

- [ ] 1. Opened a terminal, ran the two "Core" lines → saw everything **Started/Running**
- [ ] 2. (First time only) Added SIM `999700000000001` in the web page (admin / 1423)
- [ ] 3. New window → started the **tower** → saw **"NG Setup successful"**
- [ ] 4. New window → started the **phone** (with `sudo`) → saw **"PDU session … successful"**
- [ ] 5. New window → ran the **ping** → saw **"0% packet loss"** ✅

**Shutting down**

- [ ] 6a. Phone window → **Ctrl+C**
- [ ] 6b. Tower window → **Ctrl+C**
- [ ] 6c. Core window → `docker compose -f sa-deploy.yaml down`

**Golden rules**

- One character per window (Core, tower, phone, test = four windows).
- Leave the tower and phone windows open while you're using the lab.
- Type commands exactly; spaces matter; paste with **Ctrl+Shift+V**.

\newpage

# If something goes wrong (simple fixes)

Find the thing you're seeing in the left column and do what's on the right. Try these before asking
anyone — they cover almost everything.

| What you see | What it means | What to do |
|--------------|---------------|------------|
| Red text / "error" right after Step 1 | The network didn't start cleanly | Wait 20 seconds, then run the Step 1 "up" line again. If it still fails, run the shutdown line (`… down`) and then the "up" line once more. |
| Tower says **"no cells in coverage"** or exits at once | The tower and phone were started in the **same** window, so the tower shut off | Close it, start the **tower** in its own window (Step 3), wait for "NG Setup successful," *then* start the phone in a **separate** window. |
| Phone says **"MAC failure"** or **"authentication reject"** | The SIM isn't registered, or the wrong key | Check you added SIM `999700000000001` in the web page (Step 2). |
| Ping says **"100% packet loss"** | Phone connected but data path isn't up | Stop the phone (Ctrl+C) and start it again (Step 4). If still failing, restart the Core (shutdown line, then the Step 1 "up" line), then redo Steps 3–5. |
| **"connect: Network is unreachable"** on ping | You forgot the `-I uesimtun0` part | Type the ping line again exactly, including `-I uesimtun0`. |
| Terminal says **"command not found"** | A typo, or you're in the wrong folder | Retype the line carefully. Make sure you ran the `cd …` line first so you're in the right folder. |
| A window is stuck / you want to stop something | | Click that window and press **Ctrl+C**. This safely stops whatever is running in it. |
| You closed the tower or phone window by mistake | That character switched off | Just start it again in a fresh window (Step 3 for the tower, Step 4 for the phone). |

**Still stuck?** There is a helper file called **`5G_Lab_Diagnostic_Tool.xlsm`**. Open it, paste the
red error message you saw into the yellow box, and it tells you what it means and how to fix it.

\newpage

# A tiny glossary (only if you're curious)

You do **not** need any of this to use the lab. It's here in case you want to know what the words
mean.

- **UE** — the pretend **phone**.
- **gNB** — the pretend **5G cell tower**.
- **Core** — the **network brain**: checks the SIM, allows the phone on, connects it to the internet.
- **SIM / IMSI** — the phone's **identity card and its number** (`999700000000001` here).
- **Terminal** — the **black window** where you type commands.
- **Ping** — a **"are you there?" test** to the internet; `0% packet loss` means "yes, perfectly."
- **`uesimtun0`** — the name of the **phone's internet connection** on the computer.
- **Docker / Compose** — the tool that runs all the network pieces together; you just type the two
  lines and it handles the rest.

**The one thing to remember:** Core on → tower on → phone on → test → (later) all off. Everything
else is detail.
