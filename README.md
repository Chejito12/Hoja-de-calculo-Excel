[SergioGonzález.html](https://github.com/user-attachments/files/32271503/SergioGonzalez.html)
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hoja de cálculo</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }

        body {
            font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            padding: 8px;
            background: #f8f9fa;
        }

        header {
            display: flex;
            align-items: center;
            gap: 10px;
            margin-bottom: 6px;
        }
        header img { max-width: 20px; height: auto; }
        header h1  { font-size: 15px; color: #333; }

        .toolbar {
            display: flex;
            gap: 6px;
            margin-bottom: 6px;
        }
        button {
            padding: 3px 10px;
            font-size: 12px;
            cursor: pointer;
            border: 1px solid #bbb;
            background: #fff;
            border-radius: 3px;
        }
        button:hover { background: #e8e8e8; }

        .formula-bar {
            display: flex;
            align-items: center;
            gap: 6px;
            margin-bottom: 5px;
        }
        #cell-name {
            width: 44px;
            text-align: center;
            border: 1px solid #ccc;
            padding: 2px 4px;
            background: #eee;
            font-size: 12px;
        }
        #formula-input {
            flex: 1;
            border: 1px solid #ccc;
            padding: 2px 6px;
            font-size: 12px;
            background: #fff;
            font-family: monospace;
        }

        table {
            border-collapse: collapse;
            background: #fff;
            box-shadow: 0 1px 4px rgba(0,0,0,.12);
        }

        th {
            border: 1px solid #ccc;
            font-weight: normal;
            font-size: 11px;
            text-align: center;
            height: 20px;
            padding: 0 4px;
            user-select: none;
            background: #eee;
        }
        th.col-h {
            width: 80px;
            cursor: pointer;
        }
        th.col-h:hover { background: #d5d5d5; }
        th.num-h { width: 32px; }

        td {
            border: 1px solid #ddd;
            font-size: 12px;
            width: 80px;
            height: 22px;
            position: relative;
            padding: 0;
        }
        td.num-h {
            background: #eee;
            border-color: #ccc;
            text-align: center;
            font-size: 11px;
            color: #666;
            user-select: none;
            width: 32px;
        }

        .cv {
            position: absolute;
            inset: 0;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 0 4px;
            overflow: hidden;
            white-space: nowrap;
            pointer-events: none;
        }

        .ci {
            position: absolute;
            inset: 0;
            border: 0;
            opacity: 0;
            pointer-events: none;
            width: 100%;
            height: 100%;
            font-size: 12px;
            text-align: center;
            background: transparent;
            font-family: monospace;
        }
        .ci:focus {
            opacity: 1;
            outline: 2px solid #1a73e8;
            outline-offset: -2px;
            background: #fff;
            z-index: 2;
            pointer-events: auto;
        }

        .neg  { color: #c0392b; font-weight: 600; }
        .err  { color: #e74c3c; font-size: 10px; font-style: italic; }

        td.active { outline: 2px solid #1a73e8; outline-offset: -2px; z-index: 1; }

        td.col-sel  { background: #e8f0fe !important; }
        th.col-sel  { background: #4285f4 !important; color: #fff; }

        #status {
            font-size: 11px;
            color: #666;
            margin-top: 5px;
            min-height: 14px;
        }
    </style>
</head>
<body>

<header>
    <img src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSJWeABAmhV12Opm1jZmowmC4qXr4_-G8vnPoG2keoC6Q&s=10" alt="logo"/>
    <h1>Hoja de cálculo</h1>
</header>

<div class="toolbar">
    <button id="btn-save">Guardar</button>
    <button id="btn-csv">Exportar</button>
    <button id="btn-new">Hoja Nueva</button>
</div>

<div class="formula-bar">
    <span id="cell-name">—</span>
    <input id="formula-input" type="text" placeholder="Selecciona una celda" readonly />
</div>

<table>
    <thead></thead>
    <tbody></tbody>
</table>

<div id="status">Hoja1</div>

<script>
const ROWS        = 20
const COLS        = 25
const FIRST       = 65          
const STORAGE_KEY = 'hojaclara_v1'

const $  = s => document.querySelector(s)
const $$ = s => document.querySelectorAll(s)
const range = n => Array.from({ length: n }, (_, i) => i)
const colLetter = i => String.fromCharCode(FIRST + i)
const clamp = (v, lo, hi) => Math.max(lo, Math.min(hi, v))   

function parseRef(ref) {
    const m = ref.match(/^([A-Z]+)(\d+)$/)
    if (!m) return null
    const col = m[1].charCodeAt(0) - FIRST
    const row = parseInt(m[2]) - 1
    if (col < 0 || col >= COLS || row < 0 || row >= ROWS) return null
    return { col, row }
}

function escAttr(s) {
    return String(s).replace(/&/g, '&amp;').replace(/"/g, '&quot;')
}

function setStatus(msg) {
    $('#status').textContent = msg
    clearTimeout(setStatus._t)
    setStatus._t = setTimeout(() => { $('#status').textContent = 'Lista' }, 3000)
}

const FUNC_NAMES = ['SUMA', 'PROMEDIO', 'MAX', 'MIN']

function tokenize(formula) {
    const tokens = []
    let i = 0

    while (i < formula.length) {
        const ch = formula[i]

        if (ch === ' ' || ch === '\t') { i++; continue }

        if (/\d/.test(ch) || (ch === '.' && /\d/.test(formula[i + 1] ?? ''))) {
            let s = ''
            while (i < formula.length && /[\d.]/.test(formula[i])) s += formula[i++]
            tokens.push({ t: 'NUM', v: parseFloat(s) })
            continue
        }

        if (/[A-Za-z]/.test(ch)) {
            let name = ''
            while (i < formula.length && /[A-Za-z]/.test(formula[i])) name += formula[i++].toUpperCase()

            if (/\d/.test(formula[i] ?? '')) {
                let num = ''
                while (i < formula.length && /\d/.test(formula[i])) num += formula[i++]
                tokens.push({ t: 'REF', v: name + num })
            } else if (FUNC_NAMES.includes(name)) {
                tokens.push({ t: 'FUN', v: name })
            } else {
                throw new Error('#ERROR!')
            }
            continue
        }

        if ('+-*/'.includes(ch)) { tokens.push({ t: 'OP',  v: ch }); i++; continue }
        if (ch === '(')          { tokens.push({ t: 'LP'           }); i++; continue }
        if (ch === ')')          { tokens.push({ t: 'RP'           }); i++; continue }
        if (ch === ':')          { tokens.push({ t: 'COL'          }); i++; continue }
        if (ch === ',')          { tokens.push({ t: 'COM'          }); i++; continue }

        throw new Error('#ERROR!')
    }

    return tokens
}

function parseFormula(tokens, state) {
    let pos = 0

    const peek    = () => tokens[pos]
    const consume = () => tokens[pos++]
    const expect  = type => {
        const tok = consume()
        if (!tok || tok.t !== type) throw new Error('#ERROR!')
        return tok
    }

    function expr() {
        let left = term()
        while (peek()?.t === 'OP' && (peek().v === '+' || peek().v === '-')) {
            const op = consume().v
            const right = term()
            left = op === '+' ? left + right : left - right
        }
        return left
    }

    function term() {
        let left = factor()
        while (peek()?.t === 'OP' && (peek().v === '*' || peek().v === '/')) {
            const op = consume().v
            const right = factor()
            if (op === '/') {
                if (right === 0) throw new Error('#DIV/0!')
                left = left / right
            } else {
                left = left * right
            }
        }
        return left
    }

    function factor() {
        const tok = peek()
        if (!tok) throw new Error('#ERROR!')

        if (tok.t === 'OP' && tok.v === '-') { consume(); return -factor() }
        if (tok.t === 'OP' && tok.v === '+') { consume(); return +factor() }

        if (tok.t === 'LP') {
            consume()
            const v = expr()
            expect('RP')
            return v
        }

        if (tok.t === 'FUN') return funcCall()

        if (tok.t === 'REF') {
            consume()
            return resolveRef(tok.v, state)
        }

        if (tok.t === 'NUM') {
            consume()
            return tok.v
        }

        throw new Error('#ERROR!')
    }

    function funcCall() {
        const fname = consume().v
        expect('LP')
        const startTok = consume()
        if (startTok?.t !== 'REF') throw new Error('#ERROR!')
        expect('COL')
        const endTok = consume()
        if (endTok?.t !== 'REF') throw new Error('#ERROR!')
        expect('RP')

        const vals = rangeValues(startTok.v, endTok.v, state)
        if (!vals.length) return 0

        switch (fname) {
            case 'SUMA':     return vals.reduce((a, b) => a + b, 0)
            case 'PROMEDIO': return vals.reduce((a, b) => a + b, 0) / vals.length
            case 'MAX':      return Math.max(...vals)
            case 'MIN':      return Math.min(...vals)
            default:         throw new Error('#ERROR!')
        }
    }

    const result = expr()
    if (pos < tokens.length) throw new Error('#ERROR!')
    return result
}

const computing = new Set()

function resolveRef(ref, state) {
    const pos = parseRef(ref)
    if (!pos) throw new Error('#REF!')
    const cell = state[pos.col][pos.row]
    const val  = evalCell(ref, cell.value, state)
    if (typeof val === 'string' && val.startsWith('#')) throw new Error(val)
    return typeof val === 'number' ? val : 0
}

function rangeValues(startRef, endRef, state) {
    const s = parseRef(startRef)
    const e = parseRef(endRef)
    if (!s || !e) throw new Error('#REF!')

    const vals = []
    const c0 = Math.min(s.col, e.col), c1 = Math.max(s.col, e.col)
    const r0 = Math.min(s.row, e.row), r1 = Math.max(s.row, e.row)

    for (let col = c0; col <= c1; col++) {
        for (let row = r0; row <= r1; row++) {
            const ref  = `${colLetter(col)}${row + 1}`
            const cell = state[col][row]
            const val  = evalCell(ref, cell.value, state)
            if (typeof val === 'string' && val.startsWith('#')) throw new Error(val)
            if (typeof val === 'number') vals.push(val)
        }
    }
    return vals
}

function evalCell(cellId, value, state) {
    if (computing.has(cellId)) return '#CIRCULAR!'
    if (value === '' || value == null) return 0
    if (typeof value === 'number') return value
    if (!value.startsWith('=')) {
        const n = parseFloat(value)
        return isNaN(n) ? value : n
    }
    const formula = value.slice(1).trim()
    if (!formula) return 0
    computing.add(cellId)
    try {
        const tokens = tokenize(formula)
        return parseFormula(tokens, state)
    } catch (e) {
        return e.message.startsWith('#') ? e.message : '#ERROR!'
    } finally {
        computing.delete(cellId)
    }
}

let STATE = range(COLS).map(() =>
    range(ROWS).map(() => ({ value: '', computedValue: 0 }))
)
let activeCell = null
let activeCol  = null   

function computeAll(state) {
    for (let pass = 0; pass < 2; pass++) {
        state.forEach((col, x) =>
            col.forEach((cell, y) => {
                const id = `${colLetter(x)}${y + 1}`
                cell.computedValue = evalCell(id, cell.value, state)
            })
        )
    }
}

function updateCell({ x, y, value }) {
    const s = structuredClone(STATE)
    s[x][y].value = value
    computeAll(s)
    STATE = s
    autoSave()
    render()
}

function autoSave() {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(STATE))
}

function loadSaved() {
    try {
        const raw = localStorage.getItem(STORAGE_KEY)
        if (!raw) return
        const saved = JSON.parse(raw)
        if (Array.isArray(saved) && saved.length === COLS && saved[0]?.length === ROWS) {
            STATE = saved
            setStatus('Sesion anterior restaurada')
        }
    } catch (_) {  }
}

function exportCSV() {
    const rows = range(ROWS).map(row =>
        range(COLS).map(col => {
            const v = String(STATE[col][row].computedValue)
            return v.includes(',') || v.includes('"') ? `"${v.replace(/"/g, '""')}"` : v
        }).join(',')
    )
    const blob = new Blob([rows.join('\r\n')], { type: 'text/csv;charset=utf-8;' })
    const a = Object.assign(document.createElement('a'), {
        href: URL.createObjectURL(blob),
        download: 'hojaclara.csv'
    })
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    URL.revokeObjectURL(a.href)
    setStatus('Exportado como hojaclara.csv')
}

function clearSheet() {
    if (!confirm('Limpiar toda la hoja? No se puede deshacer.')) return
    STATE = range(COLS).map(() =>
        range(ROWS).map(() => ({ value: '', computedValue: 0 }))
    )
    localStorage.removeItem(STORAGE_KEY)
    activeCell = null
    activeCol  = null
    render()
    setStatus('Hoja nueva')
}

function render() {
    $('thead').innerHTML = `<tr>
        <th class="num-h"></th>
        ${range(COLS).map(i =>
            `<th class="col-h${activeCol === i ? ' col-sel' : ''}" data-col="${i}">${colLetter(i)}</th>`
        ).join('')}
    </tr>`

    $('tbody').innerHTML = range(ROWS).map(row =>
        `<tr>
            <td class="num-h">${row + 1}</td>
            ${range(COLS).map(col => {
                const cell  = STATE[col][row]
                const v     = cell.computedValue
                const isErr = typeof v === 'string' && v.startsWith('#')
                const isNeg = typeof v === 'number'  && v < 0        
                const disp  = (v === 0 && cell.value === '') ? '' : v
                const spanCls = isErr ? 'cv err' : isNeg ? 'cv neg' : 'cv'
                const isAct = activeCell?.x === col && activeCell?.y === row
                const isCS  = activeCol === col && !isAct            
                const tdCls = [isAct ? 'active' : '', isCS ? 'col-sel' : ''].filter(Boolean).join(' ')
                return `<td data-x="${col}" data-y="${row}" class="${tdCls}">
                    <span class="${spanCls}">${disp}</span>
                    <input class="ci" type="text" value="${escAttr(cell.value)}" />
                </td>`
            }).join('')}
        </tr>`
    ).join('')
}

function activateCell(x, y) {
    activeCell = { x, y }
    activeCol  = null      
    render()

    const td    = $(`td[data-x="${x}"][data-y="${y}"]`)
    const input = td?.querySelector('.ci')
    if (!input) return

    $('#cell-name').textContent  = `${colLetter(x)}${y + 1}`
    $('#formula-input').value    = STATE[x][y].value
    $('#formula-input').readOnly = true

    input.style.opacity       = '1'
    input.style.pointerEvents = 'auto'
    input.focus()
    input.setSelectionRange(input.value.length, input.value.length)

    let navDest = null

    const onInput = () => { $('#formula-input').value = input.value }

    const onKey = (e) => {
        let dest = null

        switch (e.key) {
            case 'Escape':
                input.value = STATE[x][y].value
                input.blur()
                return

            case 'Enter':
                dest = { nx: x,   ny: y + 1 }
                e.preventDefault()
                break
            case 'Tab':
                dest = { nx: x + 1, ny: y }
                e.preventDefault()
                break
            case 'ArrowUp':
                dest = { nx: x, ny: y - 1 }
                e.preventDefault()
                break
            case 'ArrowDown':
                dest = { nx: x, ny: y + 1 }
                e.preventDefault()
                break
            case 'ArrowLeft':
                if (input.selectionStart === 0) {
                    dest = { nx: x - 1, ny: y }
                    e.preventDefault()
                }
                break
            case 'ArrowRight':
                if (input.selectionStart === input.value.length) {
                    dest = { nx: x + 1, ny: y }
                    e.preventDefault()
                }
                break
        }

        if (dest) {
            navDest = {
                nx: clamp(dest.nx, 0, COLS - 1),
                ny: clamp(dest.ny, 0, ROWS - 1)
            }
            input.blur()
        }
    }

    const onBlur = () => {
        input.removeEventListener('keydown', onKey)
        input.removeEventListener('input',   onInput)
        input.style.opacity       = '0'
        input.style.pointerEvents = 'none'

        if (input.value !== STATE[x][y].value) {
            updateCell({ x, y, value: input.value })
        }

        if (navDest) {
            const { nx, ny } = navDest
            navDest = null
            setTimeout(() => activateCell(nx, ny), 0)
        }
    }

    input.addEventListener('keydown', onKey)
    input.addEventListener('input',   onInput)
    input.addEventListener('blur',    onBlur, { once: true })
}

function setupEvents() {
    $('tbody').addEventListener('click', e => {
        const td = e.target.closest('td[data-x]')
        if (!td) return
        activateCell(Number(td.dataset.x), Number(td.dataset.y))
    })

    $('thead').addEventListener('click', e => {
        const th = e.target.closest('th[data-col]')
        if (!th) return
        activeCol  = Number(th.dataset.col)
        activeCell = null
        render()
    })

    document.addEventListener('keydown', e => {
        if (e.key === 'Backspace' && activeCol !== null && document.activeElement.tagName !== 'INPUT') {
            range(ROWS).forEach(row => updateCell({ x: activeCol, y: row, value: '' }))
            setStatus(`Columna ${colLetter(activeCol)} borrada`)
        }
    })

    document.addEventListener('copy', e => {
        if (activeCol !== null) {
            const vals = range(ROWS).map(r => STATE[activeCol][r].computedValue)
            e.clipboardData.setData('text/plain', vals.join('\n'))
            e.preventDefault()
            setStatus('Columna copiada al portapapeles')
        }
    })

    $('#btn-save').addEventListener('click', () => { autoSave(); setStatus('Guardado') })
    $('#btn-csv').addEventListener('click',  exportCSV)
    $('#btn-new').addEventListener('click',  clearSheet)
}

loadSaved()
render()
setupEvents()
</script>
<script type="module" src="https://static.cloudflareinsights.com/beacon.min.js/v31edd6df95cf4e85bb4c19e7a9bdbcba1788362987495" integrity="sha512-iIg7k2xntmwu6/uSb5tpc/hySgZc4eoL31yB29W6tJFo2akwjPWcEqnCEdJvGexCL0KEQwVYv5BlowfhVz26hg==" data-cf-beacon='{"version":"2024.11.0","token":"f0f03d0605bf45f1a3b17cf7d5e6861f","spa":2}' crossorigin="anonymous"></script>
</body>
</html>
