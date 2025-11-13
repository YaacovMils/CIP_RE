# Definitive "As-Made" Analysis for CIP Process A

This document provides a definitive, traceable analysis of the "As-Made" Cleaning-in-Place (CIP) process for Line A. Every logical statement is cited with its precise code location `(File, Rung)` for verification.

---
## External Interfaces & Communication

This process communicates with external systems in several key ways:

*   **HMI/SCADA:** The primary control and status feedback loop is with an HMI/SCADA system.
    *   **Inputs from HMI:** `CIPACMD`, `AMMISTP`, `STOPALL`, `ENDALL`.
    *   **Outputs to HMI:** `A_STAT` (a compiled status word), `ALRM_96` (an aggregated alarm word), `R09060` (stop status), and `FQCAQTY` (flow totalizer).
*   **Genius Bus:** The flow totalizer value, `FQCAQTY`, is explicitly noted as being sent via a "genius send," which is a GE Genius Bus communication protocol **(A_CIP, VAR_GLOBAL)**.
*   **Shared Resources (Inter-Line):** The process is interlocked with other CIP lines and shared utilities. It checks `T*_BUSY` flags (e.g., `T1_BUSY`, `T2_BUSY`) before using a shared tank. These flags are set by other processes, preventing resource contention **(A_CIP, 5)**.

---

## State 1: Pre-Rinse (`A1_STRT`)

This state performs a pre-rinse of the reactor, typically with recovered water.

*   **Entry Conditions:**
    *   The `CIPACMD` register (from HMI) is set to `1` **(A_CMD, 4)**.
    *   No other stage (`A2_STRT`, `A3_STRT`) is active **(A_CMD, 16)**.
    *   The system is not in a global stop state (`A_STOP` is FALSE) **(A_CMD, 16)**.

*   **Activations:**
    *   The `A1_STRT` flag is set to TRUE **(A_CMD, 16)**.
    *   The flow meter totalizer (`FQCAQTY`) is reset to `0` **(A_CMD, 14)**. This value is sent to the HMI.
    *   The reactor filling process (`A_FILLS`) is initiated **(A_CIP, 4)**.
    *   The main supply valve to Line A reactors (`XVM047`) is opened **(A_CIP, 17)**.
    *   The CIP A supply pump (`P4M69`) is started after a 5-second delay **(A_CIP, 18)**.
    *   The supply valve from the Recovery Water Tank (`XVM036`) is opened **(A_CIP, 15)**.
    *   If the Recovery Water Tank is busy (`T2_BUSY`), the supply valve from the Fresh Water Tank (`XVM025`) is opened instead **(A_CIP, 14)**.
    *   **Inactive Logic:** The intended logic to open the return valve (`XVM008`) to the Recovery Water Tank is **permanently disabled** by an `_ALW_OFF` flag. This water-saving feature cannot function during this stage **(A_CIP, 23)**.

*   **Exit Conditions:**
    *   The `A1_STRT` flag is reset if the value of `CIPACMD` (from HMI) is not equal to `1` **(A_CMD, 9)**.

*   **Interlocks:**
    *   The entire stage is inhibited if `A_STOP` is TRUE **(A_CMD, 16)**. `A_STOP` is activated by `STOPALL` (global), `A_ALARM`, or `AMMISTP` (from HMI) **(A_CMD, 21)**.
    *   The opening of supply valves is prevented if the `CANFILL` condition is FALSE (e.g., shared tanks are busy) **(A_CIP, 5)**.

---

## State 2: Chemical Wash (`A2_STRT`)

This state performs a chemical wash using either soda or acid.

*   **Entry Conditions:**
    *   `CIPACMD` (from HMI) is `2` (Soda) AND `M00207` is TRUE, OR `CIPACMD` is `3` (Acid) AND `M00206` is TRUE **(A_CMD, 5, 6)**.
    *   No other stage (`A1_STRT`, `A3_STRT`) is active **(A_CMD, 18)**.
    *   The system is not in a global stop state (`A_STOP` is FALSE) **(A_CMD, 18)**.

*   **Activations:**
    *   The `A2_STRT` flag is set to TRUE **(A_CMD, 18)**.
    *   The `A2USE_S` flag is set to select Soda (TRUE) or Acid (FALSE) **(A_CMD, 18)**.
    *   The appropriate chemical supply valve is opened (`XVM028`, `XVM031`, or `XVM034`) **(A_CIP, 16)**.
    *   Return valves are opened (`XVM002`, `XVM004`, or `XVM006`) if conductivity is within spec **(A_CIP, 24)**.

*   **Exit Conditions:**
    *   The `A2_STRT` flag is reset if `CIPACMD` (from HMI) is not `2` (for soda) or not `3` (for acid) **(A_CMD, 10, 11)**.
    *   **Logical Flaw:** The primary exit condition for the fill process (`A_FILLS`) relies on the `A_FULL` flag. The logic to set `A_FULL` based on volume (`FQCAQTY > CIPAVOL`) is **permanently disabled** by an `_ALW_OFF` flag. The fill will only terminate based on a secondary, time-based mechanism, leading to inaccurate fills **(A_CIP, 7)**.

*   **Interlocks:**
    *   A busy shared tank (`T*_BUSY`) prevents the fill from starting **(A_CIP, 5)**.
    *   A high level in a return tank (`LSH*`) diverts return flow to the drain (`XVM011`) **(A_CIP, 22)**.

---

## State 3: Final Rinse (`A3_STRT`)

This state performs a final rinse, typically with fresh water.

*   **Entry Conditions:**
    *   The `CIPACMD` register (from HMI) is set to `4` **(A_CMD, 7)**.
    *   No other stage (`A1_STRT`, `A2_STRT`) is active **(A_CMD, 20)**.
    *   The system is not in a global stop state (`A_STOP` is FALSE) **(A_CMD, 20)**.

*   **Activations:**
    *   The `A3_STRT` flag is set to TRUE **(A_CMD, 20)**.
    *   The supply valve from the Fresh Water Tank (`XVM025`) is opened **(A_CIP, 14)**.
    *   The return valve to the Recovery Water Tank (`XVM008`) is opened **(A_CIP, 23)**.

*   **Exit Conditions:**
    *   The `A3_STRT` flag is reset if `CIPACMD` (from HMI) is not equal to `4` **(A_CMD, 12)**.

*   **Interlocks:**
    *   A busy Fresh Water Tank (`T1_BUSY`) prevents the fill from starting **(A_CIP, 5)**.
    *   A high level in the Recovery Water Tank (`LSH3074`) diverts return flow to the drain (`XVM011`) **(A_CIP, 22)**.
