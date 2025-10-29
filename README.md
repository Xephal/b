console.log("QuizMaker fix chargé !");
document.addEventListener("DOMContentLoaded", () => {
  function showExplanations() {
    document.querySelectorAll(".ays_question_explanation").forEach(exp => {
      if (exp.textContent.trim().length > 0) {
        exp.style.display = "block";
        exp.style.visibility = "visible";
      }
    });
  }

  // Première tentative (au chargement)
  showExplanations();

  // Observe tout changement dans le DOM
  const obs = new MutationObserver(showExplanations);
  obs.observe(document.body, { childList: true, subtree: true });

  // Et si un clic sur "Next", "Finish" ou "Answer" survient, on ré-affiche aussi
  document.body.addEventListener("click", e => {
    if (
      e.target.matches(".ays_next, .ays_finish, .ays-answer, button") ||
      e.target.closest(".ays_buttons_div")
    ) {
      setTimeout(showExplanations, 300);
    }
  });
});