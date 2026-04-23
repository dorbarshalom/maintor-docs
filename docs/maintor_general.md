# Maintor.systems

## General

I'm building a new product called Maintor. Maintor is a web app that helps companies track and improve their machine maintenance. These companies are industrial, like recycling, food & beverage, pharma etc.

The system is all about running maintenance tasks, then creating reports based on this data. There are 2 types of maintenance tasks:

* **Planned maintenance** \- periodical maintenance performed to keep machinery from failing  
* **Breakdown maintenance** \- performed when a machine fails

## Tech stack

* The app has 3 parts:  
  * Admins frontend, desktop first (maintor-app)  
  * Technicians frontend, mobile first (maintor-engineers)  
  * Backend, running on Cloudflare workers (maintor-api) \- **this is our current project.**  
* For UI use quasar  
* For user management use Firebase. For authentication, use signup with Google, or use email & password. We will later add additional SSO  
* Desktop app includes all app functionality. Mobile however, is used for filing tickets only.  
* The database is firestore

