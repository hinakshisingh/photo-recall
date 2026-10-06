// Serverless function: receives the user's vague memory (+ answers so far),
// asks the AI to either ask ONE narrowing question or return best matches.
const photos = require("../photos.json");

const MAX_QUESTIONS = 2;
const MAX_RESULTS = 5;

function buildPrompt(query, answers, mustFinish) {
  const library = photos.map((p) => p.id + ": " + p.description).join("\n");
  const qa = answers.length
    ? answers.map((a, i) => "Question " + (i + 1) + ": " + a.question + "\nUser answer " + (i + 1) + ": " + a.answer).join("\n")
    : "(none yet)";

  return [
    "You are the search assistant inside a photo app prototype.",
    "The user is trying to find a photo they only vaguely remember.",
    "You can only use the photo descriptions below. Never invent details that are not in them.",
    "",
    "PHOTO LIBRARY (id: description):",
    library,
    "",
    "USER'S MEMORY: " + query,
    "NARROWING QUESTIONS ALREADY ASKED AND ANSWERED:",
    qa,
    "",
    "RULES:",
    "1. Treat any year the user gives as a soft hint, not a strict filter. If you show a photo from a different year, say so in the message.",
    "2. If more than " + MAX_RESULTS + " photos plausibly fit AND one short question would separate them using details that actually differ in the descriptions (setting, people, activity, season, time of day, year), return action \"ask\". Ask only ONE question at a time. Give 2 to 4 short answer options taken from the real differences between candidate photos.",
    "3. Never ask something the user already told you or already answered.",
    "4. Questions asked so far: " + answers.length + ". Maximum allowed: " + MAX_QUESTIONS + ". " +
      (mustFinish ? "The limit is reached: you MUST return action \"results\" or \"none\" now." : "Do not ask more than the maximum."),
    "5. If " + MAX_RESULTS + " or fewer photos fit, return action \"results\" without asking.",
    "6. Return action \"results\" with up to " + MAX_RESULTS + " ids, best match first. Only include photos that genuinely fit what the user said and answered.",
    "7. If nothing in the library fits, return action \"none\" and say honestly that there is no match. Do not show weak or loosely related photos as matches.",
    "8. The message must be one or two plain sentences.",
    "",
    "Reply with JSON only, in exactly one of these shapes:",
    "{\"action\":\"ask\",\"question\":\"...\",\"options\":[\"...\",\"...\"]}",
    "{\"action\":\"results\",\"ids\":[\"p01\"],\"message\":\"...\"}",
    "{\"action\":\"none\",\"message\":\"...\"}"
  ].join("\n");
}

async function callGemini(prompt) {
  const model = process.env.GEMINI_MODEL || "gemini-2.5-flash";
  const url = "https://generativelanguage.googleapis.com/v1beta/models/" + model + ":generateContent";
  const r = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json", "x-goog-api-key": process.env.GEMINI_API_KEY },
    body: JSON.stringify({
      contents: [{ role: "user", parts: [{ text: prompt }] }],
      generationConfig: { responseMimeType: "application/json", temperature: 0.2 }
    })
  });
  if (!r.ok) throw new Error("Gemini error " + r.status + ": " + (await r.text()).slice(0, 300));
  const d = await r.json();
  const parts = (d.candidates && d.candidates[0] && d.candidates[0].content && d.candidates[0].content.parts) || [];
  return parts.map((p) => p.text || "").join("");
}

async function callGroq(prompt) {
  const model = process.env.GROQ_MODEL || "llama-3.3-70b-versatile";
  const r = await fetch("https://api.groq.com/openai/v1/chat/completions", {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: "Bearer " + process.env.GROQ_API_KEY },
    body: JSON.stringify({
      model: model,
      temperature: 0.2,
      response_format: { type: "json_object" },
      messages: [{ role: "user", content: prompt }]
    })
  });
  if (!r.ok) throw new Error("Groq error " + r.status + ": " + (await r.text()).slice(0, 300));
  const d = await r.json();
  return d.choices[0].message.content;
}

async function askModel(prompt) {
  let text;
  if (process.env.GEMINI_API_KEY) text = await callGemini(prompt);
  else if (process.env.GROQ_API_KEY) text = await callGroq(prompt);
  else throw new Error("No API key found. Add GEMINI_API_KEY or GROQ_API_KEY in Vercel settings, then redeploy.");
  const clean = text.replace(/```json|```/g, "").trim();
  return JSON.parse(clean);
}

module.exports = async (req, res) => {
  if (req.method !== "POST") {
    res.status(405).json({ error: "Use POST" });
    return;
  }
  try {
    const body = req.body || {};
    const query = String(body.query || "").slice(0, 300).trim();
    const answers = (Array.isArray(body.answers) ? body.answers : [])
      .slice(0, MAX_QUESTIONS)
      .map((a) => ({
        question: String((a && a.question) || "").slice(0, 300),
        answer: String((a && a.answer) || "").slice(0, 300)
      }));

    if (!query) {
      res.status(400).json({ error: "Type what you remember about the photo." });
      return;
    }

    const mustFinish = answers.length >= MAX_QUESTIONS;
    const result = await askModel(buildPrompt(query, answers, mustFinish));

    if (result.action === "ask") {
      if (mustFinish || !result.question) {
        res.status(502).json({ error: "The AI did not return a usable answer. Please try again." });
        return;
      }
      const options = Array.isArray(result.options) ? result.options.map(String).slice(0, 4) : [];
      res.status(200).json({ action: "ask", question: String(result.question), options: options });
      return;
    }

    const valid = new Set(photos.map((p) => p.id));
    const ids = [...new Set((Array.isArray(result.ids) ? result.ids : []).map(String))]
      .filter((id) => valid.has(id))
      .slice(0, MAX_RESULTS);

    if (result.action === "results" && ids.length > 0) {
      res.status(200).json({
        action: "results",
        ids: ids,
        message: String(result.message || "Here are the best matches."),
        questionsAsked: answers.length
      });
      return;
    }

    res.status(200).json({
      action: "none",
      message: String(result.message || "I could not find a confident match in this library."),
      questionsAsked: answers.length
    });
  } catch (err) {
    res.status(500).json({ error: err.message || "Something went wrong." });
  }
};
