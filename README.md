<div
  class="dropdown mx-2"
  data-controller="weekday-filter"
>
  <button
    class="btn btn-outline-secondary dropdown-toggle"
    type="button"
    data-bs-toggle="dropdown"
  >
    <span data-weekday-filter-target="label">
      Tous les jours
    </span>
  </button>

  <div class="dropdown-menu p-2">
    {% for day, label in {
      1: 'Lundi',
      2: 'Mardi',
      3: 'Mercredi',
      4: 'Jeudi',
      5: 'Vendredi',
      6: 'Samedi',
      7: 'Dimanche'
    } %}
      <div class="form-check">
        <input
          class="form-check-input"
          type="checkbox"
          value="{{ day }}"
          id="weekday-{{ day }}"
          data-weekday-filter-target="checkbox"
          {{ stimulus_action('weekday-filter', 'toggle', 'change') }}
          {{ stimulus_action('chart', 'onPeriodChange', 'change') }}
        >
        <label class="form-check-label" for="weekday-{{ day }}">
          {{ label }}
        </label>
      </div>
    {% endfor %}
  </div>

  <!-- inputs hidden injectés ici -->
  <div data-weekday-filter-target="inputs"></div>
</div>
