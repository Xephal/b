```tsx
{/* === Base de connaissances === */}
<label htmlFor="baseConnaissance" className="text-sm font-medium text-gray-700">
  {t("baseConnaissance")}
</label>

<div className="relative">
  {/* Tags sélectionnés */}
  <div className="flex flex-wrap gap-2 mb-1">
    {selectedBaseConnaissances.map((item) => (
      <span
        key={item}
        className="flex items-center gap-1 bg-green-50 text-green-800 px-2 py-1 rounded text-xs"
      >
        {item}
        <button
          onClick={() =>
            setSelectedBaseConnaissances((prev) =>
              prev.filter((b) => b !== item)
            )
          }
          className="text-green-700 hover:text-red-600"
        >
          ×
        </button>
      </span>
    ))}
  </div>

  {/* Sélecteur principal */}
  <select
    id="baseConnaissance"
    className="w-full border rounded px-2 py-1 text-gray-900 bg-white"
    onChange={(e) => {
      const value = e.target.value
      if (value && !selectedBaseConnaissances.includes(value)) {
        setSelectedBaseConnaissances((prev) => [...prev, value])
      }
    }}
  >
    <option value="">{t("selectBaseConnaissance")}</option>
    {conversationSettings?.baseConnaissances.map((bc: any) => (
      <option key={bc.slug} value={bc.slug}>
        {bc.label}
      </option>
    ))}
  </select>
</div>