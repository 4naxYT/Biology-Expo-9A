# Idea:
## Classing:
- [Difficult] : Might be difficult to Impliment

## Steps:
- User inputs a Blood sample into the monitoring Contraption
- User inputs their Particulars (weight,hight,etc..)
- We collect info on:
  - Range of <X>
- We Compute Defitencies and Effitiencies
- Report Advice

## Range Data ex
```json
{
	"Sugar_Level (Mg/Dl)": {
		"high": {
			"Min": 100,
			"Max": 150
		},
		"Mid": {
			"Min": 71,
			"Max": 99
		},
		"Low": {
			"Min": 0,
			"Max": 70
		}
	},
	"Hemoglobin (g/Dl)": {
		"high": {
			"Min": 17.6,
			"Max": 22.0
		},
		"Mid": {
			"Min": 13.5,
			"Max": 17.5
		},
		"Low": {
			"Min": 0,
			"Max": 13.4
		}
	},
	"White Blood Cells (K/ul)": {
		"high": {
			"Min": 11.1,
			"Max": 30.0
		},
		"Mid": {
			"Min": 4.5,
			"Max": 11.0
		},
		"Low": {
			"Min": 0,
			"Max": 4.4
		}
	},
	"Platelets (K/ul)": {
		"high": {
			"Min": 451,
			"Max": 1000
		},
		"Mid": {
			"Min": 150,
			"Max": 450
		},
		"Low": {
			"Min": 0,
			"Max": 149
		}
	}
}

```
