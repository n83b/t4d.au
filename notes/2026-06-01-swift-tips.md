# SwiftUI Clean Architecture Rules

When coding utilities natively for macOS using Swift and SwiftUI, it is incredibly easy to let view files grow massive. Sticking to built-in state structures ensures the code interface stays light, maintainable, and scannable inside Nova.

## Core Directives for View Layouts

* **Keep Views Dumb:** A view should only compute layout representation. Moving complex logic, operations, and file calculations into decoupled logical structures is crucial.
* **Leverage Native Storage Wisely:** Understand the lifecycle differences between your state parameters:

| Property Wrapper | Primary Purpose | Lifetime Scope |
| :--- | :--- | :--- |
| `@State` | Transient UI flags (toggles, strings). | Destroys/recreates with the View hierarchy. |
| `@StateObject` | Complex business logic / data fetching. | Tied directly to the persistent storage container. |

## Practical Implementation Example

Avoid processing heavy actions or loading data payloads inside the view block itself. Instead, isolate the logic entirely:

```swift
import SwiftUI

struct LogMonitorView: View {
	// Keep state localized to UI representation
	@State private var isMonitoring = false
	
	var body: some View {
		VStack(alignment: .leading, spacing: 12) {
			Text("System Monitor")
				.font(.headline)
			
			Toggle("Active Monitoring", isOn: $isMonitoring)
				.toggleStyle(.switch)
		}
		.padding()
	}
}