# Business Requirements Document - Plant Care Tracker

**Author:** Jared  
**Date:** 2026-03-30  
**Version:** 1.0

## Document Purpose

This Business Requirements Document (BRD) defines the functional capabilities required for the Plant Care Tracker mobile application. These requirements represent the complete inventory of user-facing and system capabilities that must be delivered to achieve the product vision.

## Functional Requirements

### Plant Inventory Management

- FR1: Users can add a new plant to their inventory by selecting species from a database
- FR2: Users can assign a custom name to each plant in their inventory
- FR3: Users can edit plant details (name, species, custom notes)
- FR4: Users can delete a plant from their inventory
- FR5: Users can view a list of all plants in their inventory
- FR6: Users can search or filter their plant inventory by name or species

### Care Schedule Management

- FR7: System automatically configures default care schedules based on selected plant species
- FR8: Users can customize care task frequency for individual plants (watering, fertilizing, pruning)
- FR9: Users can adjust care task timing for individual plants (specific days, times)
- FR10: Users can add custom care tasks beyond default schedules
- FR11: Users can pause care schedules for individual plants (travel mode)
- FR12: Users can resume paused care schedules
- FR13: Users can view upcoming care tasks across all plants

### Reminder & Notification System

- FR14: System delivers push notifications for scheduled care tasks
- FR15: Notifications include plant-specific information (plant name, care task type)
- FR16: Users can configure notification timing preferences
- FR17: System reschedules next reminder when care action is logged
- FR18: Users can snooze or dismiss notifications
- FR19: System provides in-app notification history as fallback for missed push notifications

### Care Logging & History

- FR20: Users can log completed care actions with timestamp
- FR21: Users can backdate care action entries
- FR22: Users can view care history for individual plants
- FR23: Users can view care history across all plants
- FR24: System displays care action details (task type, date, time)

### Species Database

- FR25: System provides species database with 50+ common houseplants
- FR26: Species database includes default care schedules (watering frequency, fertilizing frequency)
- FR27: Users can browse species database during plant addition
- FR28: System caches species database locally for offline access

### Offline Mode & Data Sync

- FR29: Users can add, edit, and delete plants without network connectivity
- FR30: Users can log care actions without network connectivity
- FR31: Users can view plant schedules and care history without network connectivity
- FR32: System syncs plant inventory across user's devices when network available
- FR33: System syncs care history across user's devices when network available
- FR34: System resolves data conflicts using last-write-wins strategy

### User Onboarding

- FR35: System guides new users through initial setup
- FR36: System requests notification permissions with clear value explanation
- FR37: Users can skip onboarding and explore app freely

## Requirements Traceability

All functional requirements in this document trace back to:
- Product vision: Reliable memory system for differentiated plant care
- User journeys: Sarah (success path) and Marcus (edge case/travel)
- Success criteria: 60% 30-day retention, 80%+ reminder action rate
- MVP scope: Problem-solving MVP with pilot validation strategy

## Acceptance Criteria

Each functional requirement must be:
- Testable: Can be verified through manual or automated testing
- Complete: Fully implemented with no partial functionality
- User-facing: Delivers tangible value to end users
- Traceable: Maps to specific user journeys and success criteria

---

**Document Status:** Final  
**Next Steps:** Use this BRD as the capability contract for UX design, technical architecture, and epic/story breakdown.
