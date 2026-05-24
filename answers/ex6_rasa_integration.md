# Ex6 — Rasa structured half

## Your answer

Implemented the structured half:
- Added checks into `ActionValidateBooking` (`rasa_project/actions/actions.py`)
- Added steps validate, rejected, confirmed into `rasa_project/data/flows.yml`
- Completed `RasaStructuredHalf.run()` in `starter/rasa_half/structured_half.py`
- Completed `normalise_booking_payload` in `starter/rasa_half/validator.py`

`make ex6-real` completed session **sess_9ef835e9959c** against live Rasa. 
Structured half outcome: complete - booking confirmed by rasa (ref=**BK-7D401E9E**).

## Citations

- `starter/rasa_half/validator.py` - `normalise_booking_payload`
- `starter/rasa_half/structured_half.py` - POST + response parse
- `rasa_project/actions/actions.py` - `ActionValidateBooking`
- `rasa_project/data/flows.yml` - `confirm_booking` flow
