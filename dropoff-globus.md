# Dropoff and Globus: Sharing Data Beyond JIC

## Getting data to (and from) collaborators without emailing a zip file

**Time:** about 15 minutes
**Level:** Beginner — no prior experience needed

By the end of this tutorial, you will be able to:

- Pick the right tool for a transfer based on size and how often you'll repeat it
- Send a dataset to an external collaborator with Dropoff
- Understand how Globus collections work and why you can't share straight from HPC storage
- Create a Globus guest collection to share a folder with a collaborator

The main idea to take away:

> **OneDrive is for small shared documents, not data. For anything bigger, Dropoff handles one-off transfers up to 300 GB, and Globus handles everything larger or recurring — but Globus shares from a collection, not directly from the HPC storage, so there's one extra step: move the data to an endpoint first.**

---

## 1. Picking the right tool

| Situation | Use |
| --- | --- |
| A few small files, sharing with internal colleagues | OneDrive / SharePoint |
| A one-off dataset up to 300 GB, internal or external | [Dropoff](https://dropoff.nbi.ac.uk/) |
| Large datasets (no practical size limit), or transfers you'll repeat | [Globus](https://www.globus.org/) |

Neither Dropoff nor Globus is a substitute for where your data actually lives day to day — see the [JIC Network Storage](jic-network-storage.md) pill for that. These are for getting a copy of it to someone who isn't on the JIC network.

## 2. Dropoff: the quick option

Dropoff is a simple web-based transfer service, good for sending or receiving a dataset without setting anything up:

1. Go to [dropoff.nbi.ac.uk](https://dropoff.nbi.ac.uk/)
2. Upload your file(s) (up to 300 GB total)
3. Enter the recipient's email address
4. They get a link to download the data, valid for a limited time

It works the other way too: you can request a drop-off from an external collaborator who doesn't have NBI credentials, and they upload directly to you.

## 3. Globus: for the large or the recurring

Globus is a high-performance transfer platform built for exactly the kind of transfer that makes Dropoff impractical: very large datasets, or ones you'll be repeating. JIC's HPC storage is registered with Globus as a collection, so once it's set up, transfers are far more reliable than a browser upload — they resume automatically if interrupted.

The collections you'll run into:

| Collection | What it is |
| --- | --- |
| **JIC HPC Data (NBI)** | The institute's main HPC storage, registered with Globus |
| **NBI Personal Short-Term Transfer Points** | A short-term external endpoint — your own staging area outside the HPC storage |
| **A guest collection you create** | A folder you've explicitly shared with specific collaborators |

**The key thing to understand:** you can't share a folder on the HPC storage directly with an external collaborator. Globus shares from collections, not from arbitrary HPC paths, so the pattern is:

1. Transfer the data from the JIC HPC Data collection to your own short-term transfer point (or a guest collection you've created)
2. Share that collection — or a guest collection built on top of it — with your collaborator
3. They connect to the shared collection in their own Globus account and pull the data

## 4. Creating a guest collection

A guest collection is how you give a specific person or group access to a specific folder, without giving them the keys to anything else:

1. In the Globus web app, navigate to the folder you want to share (on your short-term transfer point, not directly on HPC storage)
2. Create a guest collection rooted at that folder
3. Add your collaborator's Globus identity (their institutional or ORCID-linked login) with read (or read/write) access
4. They'll see the collection appear in their own "shared with me" list

Detailed, up-to-date steps are in NBI Research Computing's Confluence documentation ("Globus — Research Computing Support") — worth checking before your first share, since the exact web UI changes from time to time.

## 5. Your sharing survival kit

| Task | Where |
| --- | --- |
| Small shared documents | OneDrive / SharePoint |
| One-off transfer, up to 300 GB | [dropoff.nbi.ac.uk](https://dropoff.nbi.ac.uk/) |
| Large or recurring transfer | [Globus](https://www.globus.org/) |
| Move HPC data onto a shareable endpoint | Transfer from JIC HPC Data → your short-term transfer point |
| Share a specific folder with a collaborator | Create a Globus guest collection, add their identity |

## Practice exercise

- [ ] Upload a small test file to Dropoff and download it using a personal (non-NBI) email account, to see the full round trip
- [ ] In Globus, transfer a small test file from the JIC HPC Data collection to your personal short-term transfer point
- [ ] Create a guest collection on that transfer point and check what access options are available before actually sharing it

---

Based on JIC Informatics' Data Management training course — see the [Data Management Guide](https://jic-datamanagement.informatics.jic.ac.uk/data-management) and the [full slide deck](https://jic-datamanagement.informatics.jic.ac.uk/data-management/presentation) for the complete session.
