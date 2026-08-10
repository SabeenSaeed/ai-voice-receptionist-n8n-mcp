const requestedDate = $('Webhook').item.json.body.date;

const hoursByDay = {
  0: null,
  1: { start: 8, end: 18 },
  2: { start: 8, end: 18 },
  3: { start: 8, end: 18 },
  4: { start: 8, end: 18 },
  5: { start: 8, end: 16 },
  6: { start: 9, end: 13 },
};

const slotMinutes = 15;
const TZ_OFFSET = '+05:00';

// Timezone-independent day-of-week calculation
const [year, month, day] = requestedDate.split('-').map(Number);
const dayOfWeek = new Date(Date.UTC(year, month - 1, day)).getUTCDay();

const hours = hoursByDay[dayOfWeek];
if (!hours) {
  return [{ json: { success: true, date: requestedDate, available_slots: [], message: "Clinic is closed on this day." } }];
}

const pad = (n) => String(n).padStart(2, '0');

const allSlots = [];
for (let h = hours.start; h < hours.end; h++) {
  for (let m = 0; m < 60; m += slotMinutes) {
    allSlots.push({
      dateObj: new Date(`${requestedDate}T${pad(h)}:${pad(m)}:00${TZ_OFFSET}`),
      label: `${pad(h)}:${pad(m)}`
    });
  }
}

const busyPeriods = items
  .filter(item => item.json.start && item.json.end)
  .map(item => ({
    start: new Date(item.json.start.dateTime || item.json.start.date),
    end: new Date(item.json.end.dateTime || item.json.end.date),
  }));

const freeSlots = allSlots.filter(slot => {
  const slotEnd = new Date(slot.dateObj.getTime() + slotMinutes * 60000);
  return !busyPeriods.some(busy => slot.dateObj < busy.end && slotEnd > busy.start);
});

return [{
  json: {
    success: true,
    date: requestedDate,
    available_slots: freeSlots.map(s => s.label),
  }
}];